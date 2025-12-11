# Tolton Seeq Engineering Notebook

Technical reports, research, and experiments by James Tolton.

## Live Site

**[https://seeq12.github.io/tolton-seeq-engineering-notebook/](https://seeq12.github.io/tolton-seeq-engineering-notebook/)**

## Adding a Report

1. Copy your HTML report to `reports/`
2. Add a link entry to `index.html`
3. Commit and push

```html
<li>
  <a href="reports/your-report.html" class="report-link">
    <div class="report-title">Report Title</div>
    <div class="report-date">2025-12-11</div>
    <div class="report-desc">Brief description.</div>
  </a>
</li>
```

## Structure

```
├── index.html      # Landing page
├── reports/        # HTML report files
├── assets/         # Shared assets (if needed)
└── README.md
```
