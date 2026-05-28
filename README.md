# se-manual-qa

Manual-QA test targets for **Sourcing & Enrichment (SE)** tickets. One subdirectory per ticket
(or per bug class). Pages are served by GitHub Pages so any UC scanner / scan-adapter / CMP
under test can crawl them by URL.

**Live**: <https://ag-uc.github.io/se-manual-qa/>

## Convention

```
se-manual-qa/
├── README.md                       # this file
├── index.html                      # landing page with nav
└── <ticket-key>/
    ├── index.html                  # the QA target (publicly scannable)
    └── README.md                   # smoke recipe + pass/fail (the test plan)
```

Each ticket directory is **self-contained** (no anchor links to sibling QA pages — sibling
links can cause the scanner to crawl outward and contaminate the signal).

## Difference from `AG-UC/se-scan-dev.github.io`

| Repo | Purpose |
| --- | --- |
| `se-scan-dev.github.io` | Stable, long-lived regression targets for the V3+ Scanner suite (`scan-e2e-tests`). Variants live until the underlying scanner behaviour changes. |
| `se-manual-qa` (this repo) | Short-lived smoke targets tied to a specific ticket / PR. Pages can be removed after the fix is verified in prod and the regression risk is captured elsewhere. |

## Targets

| Ticket | Directory | Bug class | Status |
| --- | --- | --- | --- |
| [SE-3091](https://usercentrics.atlassian.net/browse/SE-3091) | [`se-3091/`](./se-3091/) | `scan-adapter` cookie-purpose cache-aliasing | Smoke ready |
| [SE-2686](https://usercentrics.atlassian.net/browse/SE-2686) / [SE-3202](https://usercentrics.atlassian.net/browse/SE-3202) | [`se-2686/`](./se-2686/) | DEA tag-delivery-method classification (GTG / SST / THIRD_PARTY) | Smoke ready — 4 sub-targets + runbook |
