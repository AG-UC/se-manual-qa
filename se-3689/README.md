# SE-3689 — scan-tracker CDP-hang timeout guards (PR #120)

Smoke targets + runbook for QA of
[scan-tracker PR #120](https://bitbucket.org/usercentricscode/scan-tracker/pull-requests/120)
(`fix(SE-3689): guard track()'s CDP calls against main-thread hangs`).

## What the fix does, in plain words

Some websites run a broken third-party script (AccessiBe's accessibility widget) that keeps
the browser's main thread 100% busy forever. When our scanner opened such a page, several of
its internal browser calls simply never got an answer, so the whole page scan hung until an
external 10-minute timeout killed it — and the customer got an empty cookie report.

PR #120 puts a stopwatch (3–5 seconds) on each of those calls. If a call gets no answer in
time, the scanner notes a warning in the result and moves on with whatever data it has,
instead of hanging forever.

## The risk we must rule out (why this QA exists)

The stopwatches must fire **only** on genuinely stuck pages. If they fire on a page that is
merely slow or complex (heavy scripts, many iframes — but still responsive), the scanner
would silently drop data (links, cookies, storage) on healthy customer sites. `track()` runs
on **every production scan**, so a false positive here is a fleet-wide data-quality
regression.

## Targets

| Target | URL | Simulates | Expected result |
| --- | --- | --- | --- |
| `blocked/` | <https://ag-uc.github.io/se-manual-qa/se-3689/blocked/> | The AccessiBe bug: main thread pinned for 90s (override: `?blockMs=7000`) | Scan **completes** (≈30s, not 10min), `warnings[]` lists timeouts, the cookie + storage item set *before* the pin are still in the result |
| `slow-healthy/` | <https://ag-uc.github.io/se-manual-qa/se-3689/slow-healthy/> | A heavy-but-working site: ~65% CPU for 12s, iframe churn, GTM tag, late cookie | **Zero** timeout warnings; all **30 links**, all **4 cookies** (`se3689_slow_c1..c3` + `se3689_slow_late`), storage items present |
| `healthy/` | <https://ag-uc.github.io/se-manual-qa/se-3689/healthy/> | A plain fast site | Fast scan, zero warnings, 10 links, 2 cookies, 1 storage item |

## How to run (local scan-tracker, no cluster needed)

```bash
cd scan-tracker
git checkout fix/SE-3689-track-cdp-hang-timeout   # PR #120 branch (main for baseline)
cd src/tracker && npm ci
npm run server        # listens on http://localhost:5051
```

Then, per target:

```bash
curl -sS -X POST http://localhost:5051/track -H "Content-Type: application/json" -d '{
  "urls": ["https://ag-uc.github.io/se-manual-qa/se-3689/slow-healthy/"],
  "minWaitTime": 50,
  "maxWaitTime": 10000,
  "enableConsent": true,
  "mergeBrowserTrackers": true
}' > result.json
```

(`mergeBrowserTrackers: true` matches production — the sidecar always sets it.)

## What to check in `result.json`

- `warnings` — the new guard warnings all contain `"timed out"`:
  `mouse simulation timed out…`, `collecting links timed out…`,
  `collectRequests: timed out detaching CDP session…`,
  `mergeBrowserTrackers: timed out retrieving browser storage state…`
- `links` — count of collected links
- `cookies` / `documentCookies` — look for the `se3689_*` marker cookies
- `storageItems` — look for the `se3689_*` marker keys

## Pass criteria

1. `blocked/` completes in well under a minute **with** timeout warnings and still returns
   the early-set cookie/storage markers.
2. `slow-healthy/` and `healthy/` complete with **zero** `timed out` warnings and the exact
   expected link/cookie/storage counts above.
3. Baseline comparison: the same scans on `main` (minus `blocked/`, which hangs there)
   return the same data as the PR branch — nothing got truncated by the change.
4. Spread check on real healthy sites (e.g. usercentrics.com, cookiebot.com, wikipedia.org,
   a slow news site): zero `timed out` warnings.
5. Known-broken real site (henryusa.com): completes with warnings instead of hanging
   (on `main` it hangs — only test this on the PR branch, and kill the request after ~60s
   if you test the baseline).
