# 100M Condition Scaling Diagnostic

**Purpose**: This document outlines the questions that must be answered BEFORE designing any scaling solution for calc-engine. Premature optimization without understanding the problem space leads to wasted effort and wrong solutions.

**Source**: Synthesized from expert consultation (Systems Engineer, Theoretical CS) and forensic architecture analysis (Glean, Slack, codebase examination).

---

## Executive Summary

### Current State
| Metric               | Value                                          |
|----------------------|------------------------------------------------|
| Largest deployment   | Intel: 870K-1.3M conditions                    |
| Practical ceiling    | ~200K monitored conditions                     |
| Architecture         | Pull-based iterators (non-resumable)           |
| Known migration path | Provider 2.0 (~70% designed, ~30% implemented) |

### Target State
| Metric           | Value            |
|------------------|------------------|
| Total conditions | 10 million       |
| Per-customer     | >1 million       |
| Timeline         | 2026             |
| Scale factor     | ~50-125x current |

### Why Questions First

1. **Bottleneck shifting**: Scaling calc-engine reveals condition-monitor as next bottleneck
2. **Unknown resource profile**: No P50/P95/P99 latency data exists
3. **Operator classification**: Don't know which operators can parallelize (CALM theorem)
4. **Root cause pattern**: Current failures stem from datasource blocking, not compute

### Critical Gaps in This Analysis

Four fundamental questions remain unaddressed and must be answered before any scaling work:

| Gap | Section | Core Question |
|-----|---------|---------------|
| **Workload Distribution** | 2.1 | Are there "celebrity" conditions/customers requiring different strategies? (The Twitter Problem) |
| **Subsystem Capacity** | 2.2 | What's the throughput ceiling of each pipeline stage? Where will it break first? |
| **Resource Optimization** | 2.10 | How much waste exists (duplicate fetches, redundant computation) that could be eliminated? |
| **Amdahl's Law** | 2.11 | What fraction of total latency is calc-engine? What's the theoretical speedup ceiling? |

**Why these are critical:**
- Without workload distribution data, we don't know if horizontal scaling works or if we need differentiated strategies
- Without per-subsystem capacity modeling, we can't predict which stage becomes the bottleneck at 100M
- Without optimization headroom analysis, we might add 10x resources when 10x waste reduction is cheaper
- Without Amdahl's Law analysis, we might optimize calc-engine when it's only 20% of the problem

---

## Part 1: Current State Assessment

### A. Architecture Overview

**Data Flow**:
```
Historian/DB → Datasource Proxy → Compute Engine (Provider)
→ Formula Evaluation (iterators) → Condition Results
→ Condition Monitor Job → Notifications
```

**Current Model Characteristics**:
- **Pull semantics**: Data fetched on-demand via `Provider` interface
- **Non-resumable**: Failures require full restart
- **Unbatched**: Individual item processing
- **Stateless**: Good for horizontal scaling but limits incremental computation
- **Conditions not partitioned**: Scheduled per monitor, not per condition

**Architectural Limitations**:
| Limitation               | Impact                                                         |
|--------------------------|----------------------------------------------------------------|
| No reified graph         | Operations primary, graph derivative - can't analyze structure |
| No monotonicity tracking | Can't classify operators as STREAMING vs BLOCKING              |
| Formulas stored as text  | Must re-parse to analyze operator properties                   |
| No windowing semantics   | Can't handle unbounded streams                                 |
| Coarse-grained caching   | No incremental cache updates                                   |

**Provider 2.0** (the designed solution):
| Aspect     | Current       | Provider 2.0   |
|------------|---------------|----------------|
| Data Flow  | Iterators     | Blocks         |
| Trigger    | Polling       | Subscriptions  |
| Execution  | On request    | As data exists |
| Recovery   | Non-resumable | Resumable      |
| Processing | Unbatched     | Batched        |

### B. Known Pain Points

**Production Failures**:

| Customer | Scale         | Issue                   | Root Cause                                          |
|----------|---------------|-------------------------|-----------------------------------------------------|
| Intel    | 870K-1.3M     | UI "basically unusable" | GET endpoint returns ALL conditions (no pagination) |
| Santos   | >10K/hour     | Performance degradation | Resource contention                                 |
| Santos   | >800 monitors | OOM                     | Memory exhaustion                                   |
| SBM      | 80K rows      | Table limit hit         | Query limits                                        |
| Multiple | Various       | OOM errors              | JVM heap + pod OOMKills                             |

**Root Cause Pattern**:
```
Slow datasource → Thread blocking (hours) → Queue buildup → System unresponsive
```

**Workarounds Currently in Production**:
- Parallel condition evaluation (shipped feature)
- Increased timeouts: 60s → 2700s
- Manual formula optimization (AE support)
- Infrastructure band-aids (memory limit bumps)
- Doubled thread pools (Flint Hills: 250K conditions working)

**Blocked Improvements**:
- Compute Scaling Enhancements: Descoped entirely
- Sample Iterator Removal: At-risk (staffing)
- Next-gen push-based calc engine: Still in planning

### C. Observability Status

**What EXISTS**:
- Signal Monitor: Per-monitor performance, distributed tracing, error logging
- Calc engine compilation timing (added Dec 2025)
- Request monitoring API (R58.0.0+)
- Infrastructure: Grafana, DataDog

**What's MISSING**:
| Gap | Impact |
|-----|--------|
| P50/P95/P99 latencies | Can't set SLAs or measure improvement |
| Conditions/sec throughput | Don't know actual capacity |
| Resource per condition | Can't identify outlier conditions |
| Queue depth metrics | Can't detect backpressure |
| Recomputation waste | Can't validate streaming migration ROI |

---

## Part 2: Operational Questions

*Questions a Senior Systems Engineer would ask before approving a 100x scale-up*

### 2.1 Workload Distribution Analysis (The Twitter Problem)

**Context from Designing Data-Intensive Applications:**

Twitter discovered two distinct user populations requiring fundamentally different architectures:
- **Celebrities**: Few users with millions of followers → read-heavy fan-out problem
- **Average users**: Many users with few followers → write-heavy, simple fan-out

A single strategy couldn't serve both efficiently. Twitter had to implement hybrid approaches with different code paths for different user types. **The same principle likely applies to calc-engine.**

**Must Answer - Customer Distribution:**
- [ ] What's the condition count distribution across customers? (Pareto 80/20?)
- [ ] Do top 10% of customers generate 90% of compute load?
- [ ] Do heavy customers have structurally different workloads (not just "more")?
- [ ] Are there customers we should route to dedicated infrastructure?
- [ ] What's the correlation between customer size and incident frequency?

**Must Answer - Condition Complexity Distribution:**
- [ ] What % are simple thresholds vs complex multi-stage formulas?
- [ ] What's the distribution of graph depth? (critical path length)
- [ ] What's the distribution of graph width? (parallelization potential)
- [ ] What % of conditions share common subgraphs?
- [ ] Are complex conditions concentrated in specific customers or spread uniformly?

**Must Answer - Hot Spot Identification:**
- [ ] Are there "hub" datasources serving thousands of conditions?
- [ ] Are there "celebrity" conditions referenced by many others?
- [ ] What's the fan-in/fan-out distribution of dependencies?
- [ ] Do hot spots correlate with production incidents?
- [ ] What's the cache hit rate distribution across datasources?

**Must Answer - Temporal Patterns:**
- [ ] What % of conditions evaluate continuously vs periodically vs ad-hoc?
- [ ] Is there time-of-day clustering? (thundering herd at shift change?)
- [ ] Do heavy customers have correlated evaluation schedules?
- [ ] What's the burstiness factor? (peak/average ratio)
- [ ] Are there predictable patterns we could exploit for scheduling?

**Strategy Implications Matrix:**

| If Distribution Is... | Strategy Implication |
|-----------------------|---------------------|
| Pareto (few heavy customers) | Dedicated infra for whales, shared pool for minnows |
| Bimodal complexity | Pre-compute complex graphs, on-demand for simple thresholds |
| Hub datasources exist | Cache aggressively at hubs, implement priority scheduling |
| Correlated bursts | Temporal sharding, staggered scheduling, admission control |
| Uniform distribution | Simple horizontal scaling works, no special cases needed |

**Red Flags Requiring Different Architecture:**
- If top 1% of conditions consume >50% of resources → outlier handling needed
- If single datasource serves >10% of conditions → single point of failure risk
- If >80% of evaluations happen in <20% of time window → capacity planning broken
- If heavy customers show different failure modes → multi-tenancy architecture issues

**Why This Matters**: Without this data, we're designing blind. Horizontal scaling assumes uniform workload. If workload is heavily skewed (likely), we need differentiated strategies—and we need to know *how* it's skewed before choosing which strategy.

### 2.2 Subsystem Capacity Model

The pipeline has distinct stages, each with independent capacity limits. Scaling one stage shifts the bottleneck to the next.

**Pipeline Architecture:**
```
┌────────────┐    ┌─────────────────┐    ┌──────────────┐    ┌───────────────────┐    ┌───────────────┐
│ Historian  │───▶│ Datasource      │───▶│ Compute      │───▶│ Condition         │───▶│ Notifications │
│ / Database │    │ Proxy           │    │ Engine       │    │ Monitor           │    │               │
└────────────┘    └─────────────────┘    └──────────────┘    └───────────────────┘    └───────────────┘
      │                   │                     │                     │                      │
      ▼                   ▼                     ▼                     ▼                      ▼
   ??? req/s          ??? req/s            ??? cond/s            ??? events/s           ??? notif/s
```

**Must Answer - Per-Subsystem Throughput:**
- [ ] What's the sustained throughput capacity of each stage?
- [ ] What's the burst capacity vs sustained capacity ratio?
- [ ] Which stage is currently the bottleneck under load?
- [ ] What's the utilization % of each stage under current production load?
- [ ] What's the headroom before each stage saturates?

**Subsystem Capacity Matrix (to be filled):**

| Subsystem | Current Throughput | Max Capacity | Utilization | Scaling Model | Scaling Limit |
|-----------|-------------------|--------------|-------------|---------------|---------------|
| Historian connections | ? | ? | ? | Connection pool | Pool size? |
| Datasource proxy | ? req/s | ? | ? | Horizontal? | Network? |
| Compute engine | ? cond/s | ? | ? | Horizontal? | Memory? |
| Condition monitor | ? events/s | ? | ? | ? | State size? |
| Notification service | ? notif/s | ? | ? | ? | External API? |
| Result storage | ? writes/s | ? | ? | ? | Disk I/O? |

**Must Answer - Inter-Stage Buffering:**
- [ ] What queues/buffers exist between stages?
- [ ] What's the queue depth under normal operation? Under stress?
- [ ] What happens when a queue fills? (backpressure? drop? block upstream?)
- [ ] Can you observe queue depth in production dashboards?
- [ ] What's the memory cost of queue growth?

**Must Answer - Backpressure Mechanisms:**
- [ ] Does slow downstream propagate backpressure upstream?
- [ ] Or does upstream keep producing, causing unbounded queue growth?
- [ ] Are there circuit breakers between stages?
- [ ] What's the failure mode when backpressure mechanisms fail?
- [ ] How long until queue overflow causes OOM?

**Must Answer - Failure Domain Isolation:**
- [ ] If Datasource Proxy fails, what's the blast radius?
- [ ] If Compute Engine OOMs, does it take down other components?
- [ ] Can stages restart independently without full pipeline restart?
- [ ] Is there bulkhead isolation between tenants within each stage?
- [ ] What's the recovery time for each stage independently?

**Must Answer - Resource Contention:**
- [ ] Do stages share CPU/memory/network resources on same nodes?
- [ ] Can one stage starve another of resources?
- [ ] Are there shared locks, singletons, or coordination points?
- [ ] What's the coordination overhead vs actual computation work?
- [ ] Are there hidden synchronization bottlenecks (logging, metrics emission)?

**Capacity Planning Formula:**

At 100M conditions (50-125x current scale), each stage must either:
1. Handle 50-125x current throughput, OR
2. Scale horizontally by 50-125x, OR
3. Reduce per-condition cost by 50-125x through architectural changes

**If ANY stage cannot achieve one of these, that stage becomes the system ceiling.**

**Why This Matters**: The document currently treats the pipeline as a black box. But at 100M scale, we need to know exactly where it will break. Investing in Compute Engine optimization is wasted if Condition Monitor caps at 10M events/sec.

### 2.3 Resource Utilization Profile

**Must Answer**:
- [ ] What's the 95th/99th percentile resource utilization under current load?
- [ ] What's the breakdown: computation vs serialization vs coordination vs GC?
- [ ] What's the memory allocation rate and GC pressure profile?
- [ ] What's the data fetch:computation ratio (time fetching vs computing)?

**Why**: If already at 70%+ CPU/memory at current scale, can't make 100x without fundamental changes.

### 2.4 Load Characteristics

**Must Answer**:
- [ ] What's the distribution of graph sizes (nodes, edges, depth)?
- [ ] What's the distribution of data access patterns (hot vs cold, temporal locality)?
- [ ] What percentage of conditions share common subgraphs or data dependencies?
- [ ] What's the temporal pattern? (continuous vs periodic bursts vs ad-hoc)

**Why**: 100M tiny graphs vs 1M complex graphs = completely different bottlenecks.

### 2.5 Failure Mode Analysis

**Must Answer**:
- [ ] What's failing NOW at current scale?
- [ ] What's the tail latency distribution (p50, p95, p99, p99.9)?
- [ ] What causes restarts/crashes today? Frequency?
- [ ] Can you detect silent failures (wrong results vs no results)?

**Why**: Existing failures at 1M will be catastrophic at 100M.

### 2.6 Observability Gaps

**Must Answer**:
- [ ] Can you trace a single condition evaluation end-to-end with timing breakdown?
- [ ] Do you have resource attribution per condition/graph/tenant?
- [ ] Can you measure coordination overhead separately from work?
- [ ] Do you have leading indicators of exhaustion (queue depth, request age)?

**Why**: Can't fix what you can't see. Need early warning signals, not lagging indicators.

### 2.7 Architectural Constraints

**Must Answer**:
- [ ] What's the coordination model? (centralized scheduler vs distributed)
- [ ] How many network hops does typical evaluation require?
- [ ] What's the fanout factor? (one request → N downstream)
- [ ] Where does data get serialized/deserialized?
- [ ] Do you have back-pressure mechanisms between stages?

**Why**: Centralized coordinators hit scaling walls fast. Each hop adds latency variance.

### 2.8 Downstream Impact

**Must Answer**:
- [ ] What's condition-monitor's throughput capacity TODAY?
- [ ] What's the write capacity of storage layer?
- [ ] Can downstream consumers keep up with increased update rate?
- [ ] What's the blast radius of single node failure?
- [ ] Do you have graceful degradation modes?

**Why**: Scaling calc-engine is pointless if condition-monitor caps at 10M/sec.

### 2.9 Scaling Cliff Detection

**Must Answer**:
- [ ] How does latency change when you double current load?
- [ ] What's cache hit rate? How does it change with scale?
- [ ] Do you have global locks, singletons, or centralized coordinators?
- [ ] What happens when you restart all nodes simultaneously?
- [ ] What's memory usage growth rate vs condition count?

**Why**: If latency >2x when load doubles, you have superlinear scaling. Cliff ahead.

### 2.10 Resource Optimization Potential

Before scaling by adding resources, determine how much waste exists that could be eliminated.

**Reference**: See `resource-optimization-tutorial/` for implementation techniques.

#### Source Layer (Fetch Optimization)

**Must Answer - Fetch Deduplication:**
- [ ] What % of historian fetches are duplicates across concurrent evaluations?
- [ ] What's the fetch deduplication potential? (N graphs, M shared inputs → M fetches vs N×M)
- [ ] Is request coalescing implemented? If so, what's the hit rate?
- [ ] Are identical fetch specs being canonicalized and hashed?

**Must Answer - Interval Subsumption:**
- [ ] What % of time range requests overlap with recent/cached requests?
- [ ] Is range subsumption implemented? ([a,b) cached, [c,d) requested where [c,d) ⊆ [a,b))
- [ ] What's the gap-filling opportunity? (fetching only missing portions)
- [ ] Are requests being coalesced into larger ranges?

**Prior Art**: Hadoop rack awareness, Spark delay scheduling, SharedDB scan sharing

#### Compute Layer (Computation Optimization)

**Must Answer - Content-Addressed Computation:**
- [ ] Are computations identified by content hash? (H(op, H(input₁), H(input₂), ...))
- [ ] What % of computations are duplicated across machines/graphs?
- [ ] Is there a global computation cache? Hit rate?
- [ ] Is single-flight/claim-check pattern implemented for in-flight deduplication?

**Must Answer - Common Subexpression Elimination:**
- [ ] What % of graphs share common subexpressions?
- [ ] Is graph canonicalization implemented? (commutativity, associativity normalization)
- [ ] Is hash-consing/interning used for structural sharing?
- [ ] What's the CSE hit rate at compile time?

**Must Answer - Multi-Query Optimization:**
- [ ] Are multiple concurrent graphs analyzed together?
- [ ] Is super-graph construction implemented? (merging N graphs into unified DAG)
- [ ] What's the source sharing potential? (shared input nodes across graphs)
- [ ] Is work sharing implemented at the operator level?

**Must Answer - Unit Canonicalization:**
- [ ] Are computations that differ only by units being deduplicated?
- [ ] Is normalization to canonical units (SI) implemented?
- [ ] What's the unit conversion overhead vs deduplication benefit?

**Prior Art**: Bazel action cache, Nix derivations, TensorFlow XLA graph optimization

#### Delivery Layer (Result Optimization)

**Must Answer - Result Deduplication:**
- [ ] Are identical results being delivered multiple times to different consumers?
- [ ] Is multicast/fan-out optimized for shared results?
- [ ] What's the network overhead for result delivery vs computation?

#### Coordination Layer (Parallelization Potential)

**Must Answer - Monotonicity Profile:**
- [ ] What % of operators are monotonic? (coordination-free parallelizable)
- [ ] How many non-monotonic barriers/strata exist in typical graphs?
- [ ] What coordination overhead could be eliminated with stratification?
- [ ] Are operators annotated with algebraic properties? (decomposable, invertible, commutative)

**Prior Art**: CALM theorem, Bloom/Bud, CRDTs, Differential Dataflow

#### Topology & Placement

**Must Answer - Data Gravity:**
- [ ] Where does data originate relative to compute? (edge, region, cloud)
- [ ] What's the network topology between historians and compute engines?
- [ ] What's the data transfer cost vs compute cost ratio?
- [ ] Could computation move to data rather than data to computation?

**Prior Art**: Edge computing (KubeEdge, AWS Greengrass), data gravity principle

#### Optimization Headroom Matrix

| Layer | Waste Type | Current Measurement | Potential Savings | How to Measure |
|-------|-----------|---------------------|-------------------|----------------|
| Source | Duplicate fetches | ? | ? | Trace analysis |
| Source | Range overlap | ? | ? | Interval analysis |
| Compute | Duplicate computation | ? | ? | Hash collision rate |
| Compute | Missed CSE | ? | ? | Canonicalization audit |
| Compute | Missed MQO | ? | ? | Super-graph analysis |
| Delivery | Redundant delivery | ? | ? | Fan-out tracing |
| Coordination | Unnecessary barriers | ? | ? | Monotonicity audit |

**Why This Matters**: Before scaling by 50-125x with more resources, determine how much of current resources are wasted on redundant work. A 10x reduction in waste may be cheaper than 10x more machines.

### 2.11 Amdahl's Law Analysis Framework

**Amdahl's Law**: Speedup = 1 / ((1-P) + P/S)
- P = fraction of work that can be improved
- S = speedup factor for that fraction
- (1-P) = fraction that CANNOT be improved (the ceiling)

**The Critical Question**: If we make calc-engine infinitely fast (S → ∞), what's the maximum system speedup?

If calc-engine is only 30% of total latency, maximum speedup = 1/(1-0.3) = 1.43x. Optimizing calc-engine beyond this is wasted effort.

#### Step 1: Identify ALL Contributors

**Must Answer - Time Decomposition:**
- [ ] What's the end-to-end latency for a condition evaluation?
- [ ] What % is spent in each component? (Must sum to 100%)

| Component | % of Total Latency | Parallelizable? | Scaling Limit |
|-----------|-------------------|-----------------|---------------|
| Historian query time | ? | ? | Connection pool? |
| Network: Historian → Compute | ? | ? | Bandwidth? |
| Calc engine: Compilation | ? | Partially | ? |
| Calc engine: Execution | ? | Yes | CPU/Memory? |
| Calc engine: Serialization | ? | ? | ? |
| Network: Compute → Results | ? | ? | Bandwidth? |
| Condition monitor processing | ? | ? | ? |
| Result storage writes | ? | ? | IOPS? |
| Notification dispatch | ? | ? | External API? |
| Coordination/scheduling overhead | ? | No (serial) | Single point? |
| GC pauses | ? | No (serial) | JVM tuning? |
| Queue wait time | ? | Depends | Backpressure? |
| **TOTAL** | 100% | | |

#### Step 2: Categorize Contributors

**Serial (Non-Parallelizable) — These are the ceiling:**
- [ ] Global locks/coordinators
- [ ] Non-monotonic synchronization barriers
- [ ] Single-threaded components
- [ ] External API rate limits

**Parallelizable but with diminishing returns:**
- [ ] Components with coordination overhead
- [ ] Shared resource contention

**Fully Parallelizable:**
- [ ] Stateless computation
- [ ] Monotonic operators
- [ ] Independent data fetches

#### Step 3: Calculate Theoretical Speedup Ceiling

**If calc-engine becomes infinitely fast:**
```
Current calc-engine fraction = P_calc
Speedup_max = 1 / (1 - P_calc)

Example: If calc-engine is 40% of latency:
  Speedup_max = 1 / 0.6 = 1.67x
  No matter how much we optimize calc-engine, system can't be more than 1.67x faster
```

**Must Answer:**
- [ ] What is P_calc (calc-engine's fraction of total latency)?
- [ ] What's the theoretical speedup ceiling for calc-engine optimization?
- [ ] What becomes the NEW bottleneck after calc-engine is optimized?

#### Step 4: Derivative Analysis (dx/dt)

**The key insight**: As we reduce calc-engine's contribution, other components become larger fractions.

**Must Answer - Bottleneck Shift:**
- [ ] If calc-engine becomes 2x faster, what's the new latency breakdown?
- [ ] If calc-engine becomes 10x faster, what component becomes dominant?
- [ ] At what point does optimizing calc-engine stop being ROI-positive?

**Bottleneck Shift Projection:**

| Calc-Engine Speedup | Calc % | New Bottleneck | System Speedup |
|---------------------|--------|----------------|----------------|
| 1x (baseline) | ?% | ? | 1.0x |
| 2x | ?% | ? | ?x |
| 5x | ?% | ? | ?x |
| 10x | ?% | ? | ?x |
| ∞ (theoretical) | 0% | ? | ?x (ceiling) |

#### Step 5: Investment Decision Framework

| If dominant component is... | Investment priority |
|-----------------------------|---------------------|
| Calc-engine (>50%) | Optimize calc-engine first |
| Network transfer (>30%) | Data locality, compression, edge computing |
| Historian queries (>30%) | Fetch deduplication, caching, connection pooling |
| Condition monitor (>30%) | Scale condition monitor BEFORE calc-engine |
| Coordination overhead (>20%) | Reduce coordination (CALM/monotonicity) |
| Serial components (>10%) | Architectural change required |

#### Drill-Down View Structure

```
TOTAL SYSTEM LATENCY (100%)
├── SOURCE LAYER (fetch + transfer)
│   ├── Historian query time
│   ├── Network: historian → compute
│   └── Fetch coordination overhead
├── COMPUTE LAYER (calc-engine)
│   ├── Compilation/parsing
│   ├── Graph traversal/scheduling
│   ├── Operator execution
│   ├── Serialization/deserialization
│   └── GC pauses
├── DELIVERY LAYER (results)
│   ├── Network: compute → downstream
│   ├── Condition monitor processing
│   ├── Result storage
│   └── Notification dispatch
└── COORDINATION LAYER (overhead)
    ├── Distributed scheduling
    ├── Non-monotonic barriers
    └── Lock contention
```

**Why This Matters**: Without this analysis, we might spend 6 months optimizing calc-engine only to discover it was 20% of the problem and condition-monitor was 50%.

---

## Part 3: Theoretical Questions

*Questions a Theoretical CS expert would ask to understand fundamental limits*

### 3.1 Fundamental Limits & Parallelism

**Must Answer**:
- [ ] What is the maximum path length (critical path) in computation graphs?
- [ ] What fraction of operations have no outgoing dependencies?
- [ ] What is the graph's width (maximum independent set at any level)?
- [ ] Are there structural bottlenecks (nodes with high in-degree/out-degree)?

**Why**: Amdahl's Law - parallel speedup bounded by longest dependency chain.

### 3.2 Monotonicity & CALM Analysis

**Must Answer**:
- [ ] Which operators produce outputs that grow monotonically with input?
- [ ] For non-monotonic operators, what triggers output retraction?
- [ ] What is the minimum number of strata needed?
- [ ] Which operations require global knowledge (argmax, ranking, quantiles)?
- [ ] What consistency guarantees are actually required?

**Why**: CALM theorem - monotonic ⟺ coordination-free. Non-monotonic requires barriers.

**CALM Implications**:
| Graph Type | Scaling Strategy |
|------------|------------------|
| Fully monotonic | Compute anywhere, merge anytime, no coordination |
| Mostly monotonic | Partition into strata, synchronize at boundaries |
| Heavily non-monotonic | Traditional distributed coordination required |

### 3.3 Complexity Cliffs

**Must Answer**:
- [ ] What is time complexity of each operator vs input size?
- [ ] Does canonicalization use AC-matching? (NP-complete)
- [ ] How does operator fusion affect complexity?
- [ ] What is fan-out complexity of broadcast nodes?
- [ ] At what N does computation change from CPU-bound to I/O-bound?

**Red Flags**:
- O(N²): Nested joins, Cartesian products
- O(2^N): Combinatorial explosion (power sets)
- AC-matching: NP-complete pattern matching

### 3.4 Operator Classification

**Must Answer**:
- [ ] Which operators are decomposable? (`f(A ∪ B) = g(f(A), f(B))`)
- [ ] Which operators are invertible? (support retraction)
- [ ] Which operator pairs commute? (order doesn't matter)
- [ ] Which operators preserve partitioning?

**Classification Matrix**:
| Operator | Decomposable | Invertible | Commutative |
|----------|--------------|------------|-------------|
| SUM | ✓ | ✓ | ✓ |
| COUNT | ✓ | ✓ | ✓ |
| MIN/MAX | ✓ | ✗ | ✓ |
| AVG | ✓ (with bookkeeping) | ✓ | ✓ |
| MEDIAN | ✗ | ✗ | ✓ |
| PERCENTILE | ✗ exact | ✗ | ✓ |

### 3.5 Information-Theoretic Bounds

**Must Answer**:
- [ ] What's minimum information that must cross stratum boundaries?
- [ ] What's communication complexity of distributed operators?
- [ ] Can operators use gossip protocols for eventual consistency?
- [ ] What's the decision tree complexity for key operations?

**Why**: Establishes theoretical best-case. If current is far from bounds, optimization possible.

---

## Part 4: Information Gaps

### What We NEED

| Category | Specific Need | Why Critical | Source |
|----------|---------------|--------------|--------|
| **Amdahl's Law** | End-to-end latency breakdown | Know calc-engine's % of total | Distributed tracing |
| **Amdahl's Law** | Per-component time attribution | Identify true bottleneck | Instrumentation |
| **Amdahl's Law** | Serial vs parallelizable classification | Calculate speedup ceiling | Architecture analysis |
| **Resource Optimization** | Fetch duplication rate | Know optimization headroom | Request tracing |
| **Resource Optimization** | Range overlap analysis | Interval subsumption potential | Time range analysis |
| **Resource Optimization** | Computation duplication rate | CSE/MQO opportunity | Hash analysis |
| **Resource Optimization** | Monotonicity ratio | Coordination reduction potential | Operator audit |
| **Workload Distribution** | Customer condition count histogram | Know if Pareto (80/20) applies | Analytics query |
| **Workload Distribution** | Condition complexity distribution | Determine if bimodal strategies needed | Formula analysis |
| **Workload Distribution** | Hot spot / hub identification | Find single points of failure | Dependency graph analysis |
| **Workload Distribution** | Temporal burst patterns | Capacity planning, scheduling | Time-series analysis |
| **Subsystem Capacity** | Per-stage throughput limits | Know which stage breaks first | Load testing |
| **Subsystem Capacity** | Queue depths between stages | Detect backpressure issues | Metrics/dashboards |
| **Subsystem Capacity** | Failure domain mapping | Understand blast radius | Architecture review |
| **Latency** | P50/P95/P99 condition evaluation | Can't set SLAs | Add instrumentation |
| **Throughput** | Conditions/sec capacity | Don't know ceiling | Add telemetry |
| **Attribution** | Resource per condition | Identify outliers | Add tracing |
| **Distribution** | Tenant size distribution | Know if 10% = 90% load | Analytics query |
| **Monotonicity** | Operator classification | Determine parallelizability | Code audit |
| **Downstream** | Condition-monitor capacity | Prevent bottleneck shift | Load test |
| **Graph Structure** | Depth, width, sharing | Understand workload | Telemetry |

### How to Get It

| Method | Reveals | Owner | Effort |
|--------|---------|-------|--------|
| **End-to-end distributed tracing** | Amdahl's Law breakdown (calc-engine %) | Platform | High |
| **Per-component time instrumentation** | True bottleneck identification | Platform | Medium |
| **Serial component audit** | Speedup ceiling calculation | Calc Engine + Platform | Medium |
| **Fetch request tracing** | Duplication rate, dedup headroom | Platform | Medium |
| **Time range overlap analysis** | Interval subsumption opportunity | Calc Engine squad | Medium |
| **Computation hash analysis** | CSE/MQO potential | Calc Engine squad | Medium |
| Customer condition histogram query | Pareto distribution, whale identification | Data team | Low |
| Formula complexity analyzer | Graph depth/width distribution | Calc Engine squad | Medium |
| Datasource dependency graph | Hub nodes, fan-in/fan-out | Calc Engine squad | Medium |
| Evaluation timestamp analysis | Temporal patterns, burst detection | Data team | Low |
| Per-stage throughput benchmarks | Subsystem capacity limits | Platform + DevEx | High |
| Queue depth instrumentation | Backpressure visibility | Platform | Medium |
| Failure injection testing | Blast radius mapping | SRE | High |
| Add P50/P95/P99 instrumentation | Latency distribution | Platform | Medium |
| Add throughput counters | Actual capacity | Platform | Low |
| Add per-condition tracing | Resource attribution | Platform | Medium |
| Analytics query on conditions | Tenant distribution | Data team | Low |
| Code audit of operators | Monotonicity classification | Calc Engine squad | High |
| Load test at 10x | Downstream capacity | DevEx | High |
| Graph analysis telemetry | Structure metrics | Calc Engine squad | Medium |

### Who to Interview

| Role | Questions | Contact |
|------|-----------|---------|
| **Platform Team** | What metrics exist? What dashboards? Current capacity estimates? | Platform lead |
| **SRE** | What incidents have occurred? What monitoring gaps? Recovery procedures? | SRE on-call |
| **Customer Success** | What do customers complain about? Workarounds in use? | CS lead |
| **Senior Engineers** | Why is architecture this way? What constraints exist? Provider 2.0 status? | Ben Johnson, Ben Bishop |
| **Product** | What's the business case for 100M? Is it validated demand? | Product manager |

---

## Part 5: Interview Guide

### Platform Team Interview

1. What telemetry currently exists for calc-engine?
2. What are current P50/P95/P99 latencies? (If unknown, why?)
3. How many conditions exist today across all tenants?
4. What's the tenant size distribution?
5. What capacity limits are documented?
6. What observability gaps do you see?
7. **[Workload]** Do you have visibility into condition complexity distribution?
8. **[Workload]** Can you identify "hot" datasources serving many conditions?
9. **[Subsystem]** What's the throughput capacity of each pipeline stage?
10. **[Subsystem]** Can you observe queue depths between stages?
11. **[Amdahl]** Can you trace end-to-end latency with per-component breakdown?
12. **[Amdahl]** What % of total latency is calc-engine vs other components?
13. **[Optimization]** What % of historian fetches are duplicates?
14. **[Optimization]** Is there a computation cache? What's the hit rate?

### SRE Interview

1. What incidents have occurred related to calc-engine/condition-monitor?
2. What's the typical failure pattern?
3. What manual interventions are required today?
4. What would break first at 10x scale?
5. What monitoring would you want before scaling?
6. **[Workload]** Do incidents correlate with specific customers or condition types?
7. **[Workload]** Are there time-of-day patterns in incidents?
8. **[Subsystem]** Which subsystem fails first under load?
9. **[Subsystem]** What's the blast radius when one component fails?
10. **[Subsystem]** Are there circuit breakers or backpressure mechanisms?
11. **[Amdahl]** When incidents occur, where is time spent? (network? compute? waiting?)
12. **[Amdahl]** What component becomes the bottleneck after you fix the primary issue?
13. **[Optimization]** Have you observed redundant work (same data fetched multiple times)?

### Customer Success Interview

1. What do customers complain about regarding conditions?
2. What workarounds do customers use?
3. Which customers have the most conditions?
4. What SLA expectations exist?
5. Are there customers we've had to refuse or limit?
6. **[Workload]** Do large customers have different usage patterns than small ones?
7. **[Workload]** Are there "power users" with unusually complex conditions?
8. **[Workload]** Do customers experience issues at specific times (shift changes)?

### Senior Engineer Interview

1. Why is the architecture pull-based?
2. What is Provider 2.0 status? Blockers?
3. Which operators are known to be problematic?
4. What assumptions are baked into current design?
5. What would you do differently if starting fresh?
6. **[Workload]** Are there known "pathological" condition patterns?
7. **[Workload]** Does the system handle heavy vs light customers differently?
8. **[Subsystem]** Which pipeline stage is currently the bottleneck?
9. **[Subsystem]** What coordination points exist between stages?
10. **[Subsystem]** What prevents independent scaling of each stage?
11. **[Amdahl]** What fraction of time is spent in calc-engine vs fetching vs delivery?
12. **[Amdahl]** If calc-engine were instant, what would be the new bottleneck?
13. **[Optimization]** Which operators are monotonic/decomposable? Is this tracked?
14. **[Optimization]** Is fetch deduplication or MQO implemented today?
15. **[Optimization]** What % of computation could be eliminated with better CSE?

---

## Part 6: Next Steps

### Immediate Actions (can do now)

- [ ] **[Amdahl - FIRST PRIORITY]** Instrument end-to-end latency with per-component breakdown
- [ ] **[Amdahl]** Calculate calc-engine's fraction of total system latency
- [ ] **[Amdahl]** Identify serial (non-parallelizable) components
- [ ] Run analytics query for tenant condition distribution
- [ ] **[Workload]** Generate customer condition count histogram (identify whales)
- [ ] **[Workload]** Analyze temporal patterns in evaluation timestamps
- [ ] Search Slack history for past scaling discussions (done - see findings)
- [ ] Review Provider 2.0 design docs in Confluence
- [ ] Schedule interviews with Platform, SRE, CS
- [ ] **[Subsystem]** Map current pipeline architecture with capacity estimates

### Short-term (requires instrumentation)

- [ ] **[Amdahl]** Build latency breakdown dashboard (per-component %)
- [ ] **[Amdahl]** Create bottleneck shift projection model
- [ ] **[Optimization]** Measure fetch duplication rate across concurrent evaluations
- [ ] **[Optimization]** Audit operators for monotonicity/decomposability properties
- [ ] **[Optimization]** Measure CSE opportunity (common subgraph analysis)
- [ ] Add P50/P95/P99 latency instrumentation
- [ ] Add throughput counters (conditions/sec)
- [ ] Add per-condition resource attribution
- [ ] Create calc-engine performance dashboard
- [ ] **[Workload]** Build formula complexity analyzer (graph depth/width)
- [ ] **[Workload]** Instrument datasource access frequency (find hubs)
- [ ] **[Subsystem]** Add queue depth metrics between pipeline stages
- [ ] **[Subsystem]** Instrument per-stage throughput and latency

### Medium-term (requires load testing)

- [ ] **[Amdahl]** Validate bottleneck shift predictions empirically
- [ ] **[Optimization]** Prototype fetch deduplication, measure savings
- [ ] **[Optimization]** Prototype MQO for concurrent graphs, measure savings
- [ ] Load test at 10x current scale
- [ ] Load test condition-monitor independently
- [ ] Profile memory usage growth curve
- [ ] Identify complexity cliffs empirically
- [ ] **[Subsystem]** Benchmark each pipeline stage independently
- [ ] **[Subsystem]** Test failure domain isolation (component failure blast radius)
- [ ] **[Workload]** Load test with synthetic "whale" customer workload

### Before Any Scaling Work

1. ✅ Document current state (this document)
2. ⬜ **[FIRST - Critical]** Complete Amdahl's Law analysis (Section 2.11) — know calc-engine's % of total latency
3. ⬜ **[Critical]** Identify theoretical speedup ceiling — know the limit before investing
4. ⬜ **[Critical]** Characterize workload distribution (Section 2.1)
5. ⬜ **[Critical]** Map subsystem capacity limits (Section 2.2)
6. ⬜ **[Critical]** Quantify resource optimization headroom (Section 2.10)
7. ⬜ Fill observability gaps (latency, throughput)
8. ⬜ Classify operators by monotonicity
9. ⬜ Load test at 10x to find cliffs
10. ⬜ Validate downstream capacity
11. ⬜ Get stakeholder alignment on priorities

---

## Appendix A: The Bottleneck Shift Problem

```
┌────────────┐    ┌─────────────┐    ┌───────────────────┐    ┌───────────────┐
│ Historian  │───▶│ Calc Engine │───▶│ Condition Monitor │───▶│ Notifications │
└────────────┘    └─────────────┘    └───────────────────┘    └───────────────┘
                        │                      │
                        ▼                      ▼
                  Scale this?           Then THIS breaks!
```

**Insight**: Optimizing calc-engine in isolation is futile if condition-monitor becomes the bottleneck.

**Questions for Pipeline Analysis**:
1. What's condition-monitor's current throughput?
2. What happens when it's overwhelmed? (Back-pressure? Silent failure?)
3. Can it scale independently?
4. What other consumers exist downstream?
5. What's the end-to-end latency budget?

---

## Appendix B: CALM Theorem Reference

**CALM**: Consistency As Logical Monotonicity

> A program has a consistent, coordination-free distributed implementation if and only if it is monotonic.

**Implications**:
- Monotonic operators (union, max, sum) → can parallelize freely
- Non-monotonic operators (negation, argmax, median) → require coordination
- Stratification partitions graph into monotonic layers with barriers between

**Operator Lattice Examples**:
| Operator | Lattice | Join (⊔) | Bottom (⊥) |
|----------|---------|----------|------------|
| MAX | (ℕ, ≤) | max | -∞ |
| MIN | (ℕ, ≥) | min | +∞ |
| SUM | (ℕ, ≤) | + | 0 |
| SET UNION | (𝒫(X), ⊆) | ∪ | ∅ |

---

## Appendix C: Source Material

- `reference-docs/brainstorming/resource-optimization-conversation.json` - Original diagnostic framework discussion
- Slack: #calc-engine-squad, #intel, #vantage-discussions-and-support - Production evidence
- Local docs: `reference-docs/calc-engine-architecture/` - Architecture analysis
- Expert consultation: Systems Engineer, Theoretical CS perspectives
- Forensic analysis: 3 architects examining Glean, Slack, codebase
