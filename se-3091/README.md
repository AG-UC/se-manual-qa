# SE-3091 — Cookiebot YSC smoke test plan

## 1. Scope

| | |
| --- | --- |
| **Ticket** | [SE-3091](https://usercentrics.atlassian.net/browse/SE-3091) / `CTS-4295` |
| **Fix in** | `scan-adapter` PR [#23304](https://dev.azure.com/cybot/Scanner%20Services/_git/scan-adapter/pullrequest/23304) — merged, on dev |
| **What's being tested** | `CookieRepositoryPurposeCacheService` no longer hands out shared mutable references that get `.Id`-rewritten on every hit |
| **Surface** | Cookie declaration for a Cookiebot dev test customer pointed at the smoke page |
| **Smoke target URL** | <https://ag-uc.github.io/se-manual-qa/se-3091/> |
| **Why only YSC** | YSC has `CookiePurposes` in **both** dev and prod Cookiebot DBs (per Michel). GA / Meta / LinkedIn / HubSpot / DoubleClick are intentionally excluded — incomplete dev DB would conflate "fix worked" with "dev has no description for that tracker". |

## 2. Preconditions

| # | Precondition |
| - | ------------ |
| 1 | Dev `scan-adapter` is running the post-fix build (PR #23304 merged + redeployed) |
| 2 | `scan-adapter` has been restarted since deploy → cache is cold (otherwise step 3 hot-cache assertion is moot) |
| 3 | Cookiebot dev has a test customer + CBID whose start URL is configured to `https://ag-uc.github.io/se-manual-qa/se-3091/` |
| 4 | Tester has access to either CB Admin / Manager UI for the test CBID, **or** can render the Cookiebot CD script in a browser |

## 3. Test steps

```
S1. Trigger a fresh scan on the test CBID.
    (force-crawl-scheduling cron tick / manual scheduler endpoint.)

S2. Wait for the scan to reach a terminal state and for scan-adapter
    post-processing to finish. Allow ~5 min depending on environment.

S3. Open the cookie declaration for the test CBID, language = English.

S4. Locate the YSC cookie entry.

S5. (Optional — hot-cache cross-scan stress) Without restarting
    scan-adapter, trigger a second scan on a different Cookiebot
    dev test customer whose start URL also loads YouTube. After
    its post-processing finishes, re-open both declarations and
    confirm YSC has a purpose on both.
```

## 4. Pass / fail

| Outcome | Verdict |
| ------- | ------- |
| YSC entry exists and its purpose field is **non-empty** in English (e.g. `"Registers a unique ID to keep statistics of what videos from YouTube the user has seen."`) | **PASS** |
| YSC entry is missing, or the purpose field is empty / null | **FAIL** — file a regression on PR #23304 |
| YSC purpose populated on first scan but missing on the second concurrent scan (step S5 only) | **FAIL** — cache aliasing still present across scans |

## 5. Out of scope

- The full 8-language translation matrix (en / da / de / es / fr / it / nl / pt-BR). This PR fixes the cache bug; missing translations are separate Cookiebot-DB content concerns the ticket conflated.
- Finnish / Swedish coverage. Same — separate content concern.
- Other major providers (GA, Meta, LinkedIn, HubSpot, DoubleClick). Their dev-DB purposes are unreliable; verify those post-prod-redeploy by retesting the customer's live declaration: `curl https://consent.cookiebot.com/48df3734-a661-4879-b14f-87d29986aa06/cd.js` and inspecting the rendered declaration in a browser.
- High-concurrency stress (≥10 concurrent scans). Step S5 covers the minimum cross-scan case.

## 6. Verification surfaces — practical notes

The public `consent.cookiebot.com/<cbid>/cd.js` is a **loader bundle** (~11 KB JS); the actual cookie data is fetched at runtime by the loader when it executes in a browser. So `curl | grep YSC` against the cd.js will **not** work for this smoke. Two options:

- **CB Admin / Manager UI** (preferred): open the test domain's cookie declaration page, find YSC, read the purpose.
- **Browser render**: build a tiny HTML page that includes `<script id="CookieDeclaration" src="https://consent.cookiebot.com/<test-cbid>/cd.js" type="text/javascript" async></script>` and open it. The rendered cookie table will list YSC with its purpose.
