# SE-2686 / SE-3202 — TagDeliveryMethod smoke test plan

## 1. Scope

| | |
| --- | --- |
| **Parent** | [SE-2686](https://usercentrics.atlassian.net/browse/SE-2686) — DEA: Add tag delivery method detection (GTG / SST / Third-party) to GCM checker |
| **QA ticket** | [SE-3202](https://usercentrics.atlassian.net/browse/SE-3202) — QA: Test TagDeliveryMethod enum detection |
| **Fix under test** | `domain-evaluation` PR [#63](https://bitbucket.org/usercentricscode/domain-evaluation/pull-requests/63) — merged 2026-05-22 |
| **Companion** | `scan-result-api` PR [#344](https://bitbucket.org/usercentricscode/scan-result-api/pull-requests/344) — wires the new fields through to scan-result-api |
| **What's being tested** | The `tagDeliveryMethod` field in DEA's GCM check correctly classifies trackers fired **prior to consent** as `GTG`, `SST`, or `THIRD_PARTY` (in scope; `UNKNOWN` is not). |

## 2. Why these pages exist

Pelle flagged on SE-3202 that "finding suitable test sites may be difficult" — real customer sites either fire trackers after consent (no detection) or have non-deterministic mixes. These four pages are **purpose-built** to fire exactly the request shape DEA looks for, **before** any consent prompt (there is no CMP on the pages at all).

Detection logic, confirmed by reading `GoogleAnalyticsRequestInterceptor.cs` at commit `9b49f6e`:

| Method | Regex matched |
| ------ | ------------- |
| `THIRD_PARTY` | `https?://[^/]*google-analytics.com/g/collect?v=` ( + `analytics.google.com`, + `google-analytics.com/g/sst`) |
| `SST` | `https?://sst.[^/]+/g/collect?v=` |
| `GTG` | Dynamically added per evaluated host: `https?://{host}/g/collect?v=` (+ `www.` variant) |
| Capture rule | **First matching request wins.** Subsequent matches are ignored. |

Critical: the interceptor hooks Playwright's `page.Request` event, which fires when the **browser initiates** the request. DNS failure / 404 / CORS doesn't matter — the URL is captured at initiation time.

## 3. Test pages

| Page | URL | What it fires | Expected `tagDeliveryMethod` |
| ---- | --- | ------------- | ---------------------------- |
| `third-party/` | <https://ag-uc.github.io/se-manual-qa/se-2686/third-party/> | `Image.src = "https://www.google-analytics.com/g/collect?v=2&..."` | `THIRD_PARTY` |
| `sst/` | <https://ag-uc.github.io/se-manual-qa/se-2686/sst/> | `Image.src = "https://sst.ag-uc.github.io/g/collect?v=2&..."` | `SST` |
| `gtg/` | <https://ag-uc.github.io/se-manual-qa/se-2686/gtg/> | `Image.src = "https://ag-uc.github.io/g/collect?v=2&..."` | `GTG` |
| `precedence/` | <https://ag-uc.github.io/se-manual-qa/se-2686/precedence/> | All three, GTG first | `GTG` (highest precedence) |

## 4. Preconditions

1. DEA dev (commit `9b49f6e` or later) is deployed and reachable at `https://domain-evaluation-api-ondemand.domain-evaluation.dev.usercentrics.cloud`.
2. You have the dev API key (`DEA_API_KEY_DEV` env var per `scan-bruno` setup; otherwise S&E Dashlane).
3. (Optional) `scan-bruno` is set up locally — already has the `Domain Evaluation API > ConsentModeCheck` request.

## 5. Test recipe

Two equivalent paths — pick one. Both POST a list of URIs to start an order, then poll the GET endpoint until the order completes.

### Option A — Bruno (recommended; the existing setup)

```
1. Open scan-bruno in Bruno.
2. Select environment: "S&E / Domain Evaluation API / On-demand DEV".
3. Open: api/v1/ConsentModeCheck/consentmodecheck.bru
4. Replace the body's `uris` with:
     [
       "https://ag-uc.github.io/se-manual-qa/se-2686/third-party/",
       "https://ag-uc.github.io/se-manual-qa/se-2686/sst/",
       "https://ag-uc.github.io/se-manual-qa/se-2686/gtg/",
       "https://ag-uc.github.io/se-manual-qa/se-2686/precedence/"
     ]
5. Send. Note the `orderId` (the post-response script extracts it into the
   GCMOrderId env var).
6. Open: api/v1/ConsentModeCheck/get-by-id.bru (or the GET request in the
   collection) and Send until status = Completed.
7. Inspect each domain's result object for tagDeliveryMethod +
   suspiciousCookiesPriorConsentDetails[].
```

### Option B — Raw curl (Swagger equivalent)

```bash
# 0. Set credentials
export DEA="https://domain-evaluation-api-ondemand.domain-evaluation.dev.usercentrics.cloud"
export API_KEY="<from Dashlane: DEA_API_KEY_DEV>"

# 1. Submit the order
ORDER=$(curl -s -X POST "$DEA/api/v1/ConsentModeCheck" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "uris": [
      "https://ag-uc.github.io/se-manual-qa/se-2686/third-party/",
      "https://ag-uc.github.io/se-manual-qa/se-2686/sst/",
      "https://ag-uc.github.io/se-manual-qa/se-2686/gtg/",
      "https://ag-uc.github.io/se-manual-qa/se-2686/precedence/"
    ]
  }' | jq -r '.orderId')

echo "orderId=$ORDER"

# 2. Poll until terminal (typically ~1-3 minutes for 4 URIs)
until curl -s "$DEA/api/v1/ConsentModeCheck?orderId=$ORDER" \
  -H "x-api-key: $API_KEY" | jq -e '.status == "Completed"' >/dev/null
do
  sleep 10
  echo "still pending …"
done

# 3. Print just the fields we care about
curl -s "$DEA/api/v1/ConsentModeCheck?orderId=$ORDER" \
  -H "x-api-key: $API_KEY" \
| jq '[.results[] | {
    uri: .uri,
    tagDeliveryMethod: .gcmConclusion.tagDeliveryMethod,
    noNonEssentialTrackersPriorConsent: .gcmConclusion.noNonEssentialTrackersPriorConsent,
    suspiciousCookiesPriorConsentDetails
  }]'
```

(The exact JSON path for `tagDeliveryMethod` may be one level different — confirm against the first response. The PR description says it lives on `GcmConclusion`.)

## 6. Pass / fail per case

| Case | Pass | Fail |
| ---- | ---- | ---- |
| `third-party/` | `tagDeliveryMethod == "THIRD_PARTY"` and a `suspiciousCookiesPriorConsentDetails[]` entry references the captured google-analytics.com request | Anything else, or empty array |
| `sst/` | `tagDeliveryMethod == "SST"` | Anything else |
| `gtg/` | `tagDeliveryMethod == "GTG"` | Anything else |
| `precedence/` | `tagDeliveryMethod == "GTG"` (happy path) | If `SST` or `THIRD_PARTY` — note as **finding**: the implementation orders by arrival, not by pattern rank; file as a follow-up under SE-2686 |

Overall **PASS** = all four cases pass.

## 7. Out of scope

- `UNKNOWN` enum value (explicitly not required by SE-3202).
- Sites with real customer GTG / SST setups. We can come back to those once dev passes; this smoke is the deterministic baseline.
- The `noNonEssentialTrackersPriorConsent` boolean and `suspiciousCookiesPriorConsentDetails` array content (`name`, `providerUrl`) — useful to spot-check during the same run but not required for SE-3202.
- The scan-result-api passthrough (`scan-result-api` PR #344) — that's a separate verification of the end-to-end V3+ flow (scan → DEA → scan-result-api → consumer). Run after this smoke passes if you want full coverage.

## 8. If something is off — quick triage

| Symptom | Most likely cause |
| ------- | ----------------- |
| All four return `UNKNOWN` | DEA dev isn't on PR #63's commit, or the api-key is hitting the wrong env. |
| `gtg/` lands as `THIRD_PARTY` | The dynamic-pattern host might not match (e.g. evaluator strips `www.` differently than expected); check `AddGtgPattern` in `GoogleAnalyticsRequestInterceptor.cs`. |
| `sst/` returns empty | The `sst.` subdomain request might have been blocked before Playwright's request event by a browser-level network policy. Try logging the DEA Handler.Runtime to see whether the request was observed. |
| `precedence/` flakes between `GTG`/`SST`/`THIRD_PARTY` | "First-arrival-wins" implementation; not a smoke failure — file as observation. |
