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
| `slow-healthy/` | <https://ag-uc.github.io/se-manual-qa/se-3689/slow-healthy/> | A heavy-but-working site: ~65% CPU for 12s, iframe churn, GTM tag, late cookie | **Zero** timeout warnings; all **30 links**, all **4 cookies** (`se3689_slow_c1..c3` + `se3689_slow_late`, written 2s in), storage items present |
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

## Real sites for the spread check (suggested, verified 2026-08-13)

AccessiBe / Cookiebot presence checked by fetching each page and grepping for
`acsbapp.com` / `consent.cookiebot.com`.

| Category | Sites | Expected |
| --- | --- | --- |
| A. Known-broken AccessiBe (CTS-4680) | henryusa.com, caesarstone.co.uk, vibrantz.com (also a Cookiebot CMP customer), magnetforensics.com (episode sometimes short-lived — zero warnings possible) | Completes with `timed out` warnings + real partial data |
| B. AccessiBe present, page healthy | accessibe.com (runs its own widget); more via BuiltWith/PublicWWW search for `acsbapp.com` | Zero warnings on most runs |
| C. Known Cookiebot customers | vaccindirekt.se (CB domain `1717503`, 39 cookies/trackers in the SE-3332 QA pass to compare against), vibrantz.com | Zero `timed out` warnings, counts comparable to earlier scans |
| D. Heavy/slow but healthy (false-positive stress) | bbc.com, cnn.com, nytimes.com, spiegel.de, bild.de, amazon.de, booking.com, aliexpress.com | Zero `timed out` warnings, plausible link/cookie counts |
| E. Fast baselines (already run) | wikipedia.org, usercentrics.com, cookiebot.com | Zero guard warnings |

Suggested order: D → C → A/B. Any unexpected warning/hang on a healthy site: diagnose with
the `hang-diagnose` skill (scan-dev-ai) before concluding.

## Executed results — 2026-08-13, PR `0fd9fb3` vs `main` `f363a90` (local, same machine)

| Target | `main` (baseline) | PR branch | Verdict |
| --- | --- | --- | --- |
| `healthy/` | 3.9s, 0 warnings, 10 links, all 3 markers | 4.8s, 0 warnings, 10 links, all 3 markers | PASS — identical |
| `slow-healthy/` | 3.7s, 0 warnings, 30 links, all 7 markers (incl. late cookie) | 4.1s, 0 warnings, 30 links, all 7 markers | PASS — identical, guards did NOT fire |
| `blocked/` | **HUNG** — killed at 90s, 0 bytes received | **28.8s**, all 5 guard warnings, pre-pin cookie + storage returned | PASS — bug reproduced on main, fixed on PR |
| henryusa.com | **HUNG** — killed at 120s, 0 bytes received | **25.8s**, 4 guard warnings, real partial data (35 tags, 79 resources) | PASS — same on the real customer site |

Guard warnings observed on the PR branch (`blocked/`): `mouse simulation timed out after 4000ms`,
`maxWaitTime of 10000ms exceeded before load-event (readyState: unknown)`,
`collecting links timed out after 3000ms`,
`collectRequests: timed out detaching CDP session after 3000ms`,
`mergeBrowserTrackers: timed out retrieving browser storage state after 5000ms — results may be incomplete`.

## Executed: real-site spread — 2026-08-14, PR `0fd9fb3` (local). Overall PASS, zero false positives

Guard warnings fired **only** where the AccessiBe bug actively pinned the thread.

| Site | Result |
| --- | --- |
| bbc.com | 8.8s, 0 warnings, 186 links, 22 cookies |
| cnn.com | 12.1s, pre-existing `waitForInactivity` only, 209 links, 32 cookies |
| nytimes.com | 12.2s, pre-existing slow-load warning (`readyState: interactive` — probe answered), 378 links, 24 cookies |
| spiegel.de | 5.7s, 0 warnings, 309 links |
| bild.de | 7.0s, 0 warnings, 656 links |
| amazon.de | 7.7s, 0 warnings, 325 links |
| de.aliexpress.com | 7.9s, 0 warnings, 180 links |
| booking.com | bot-challenge redirect → redirect error, no scan (pre-existing, unrelated) — excluded |
| vaccindirekt.se | 12.1s, pre-existing `waitForInactivity` only, 21 cookies (≈ SE-3332 baseline) |
| **www.henryusa.com** (broken) | **25.1s, 4 guard warnings, 11 cookies — completes instead of hanging** |
| **caesarstone.co.uk** (broken) | **25.5s, 4 guard warnings, 14 cookies** |
| **vibrantz.com** (broken + Cookiebot customer) | **26.7s, 4 guard warnings, 2 cookies — the production victim case now completes** |
| magnetforensics.com | 14.4s, **no** guard warnings, 151 links, 37 cookies — AccessiBe episode not active this run, full data |
| accessibe.com (widget, healthy) | 11.5s, 0 warnings, 70 links, 38 cookies |

Gotcha: `www.vibrantz.com`, apex `henryusa.com`, and `www.aliexpress.com` only 301-redirect —
the tracker treats cross-URL redirects as errors, so scan the canonical host directly.

## After merge — dev sanity + broad check (planned)

Once PR #120 is merged and the new tracker image is on dev:

1. **Sanity (broken case):** scan henryusa.com on the dev environment — expect a completed,
   non-empty result with warnings; sidecar logs no longer show 10-min
   `System.TimeoutException … GetPageTrackerResult` storms.
2. **Sanity (healthy case):** rescan 2–3 domains previously scanned on dev and compare
   cookie/tracker counts — no drops.
3. **Broad:** run a spread of regular scan targets and check results for `timed out`
   warnings — they should appear only on genuinely stuck sites. Investigate any hit with the
   `hang-diagnose` skill from `scan-dev-ai` (tells the known AccessiBe bug apart from a new hang).
4. **Observability:** watch tracker logs/Datadog for the new guard warnings' rate and confirm
   no `unhandledRejection` pod crashes.
