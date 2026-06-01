# SE-2770 — `httpRequestInjections` smoke test plan

## 1. Scope

|                              |                                                                                                                                                                                                |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ticket**                   | [SE-2770](https://usercentrics.atlassian.net/browse/SE-2770) (epic [SE-1840](https://usercentrics.atlassian.net/browse/SE-1840))                                                                |
| **Fix in (scan-config-api)** | PR [#125](https://bitbucket.org/usercentricscode/scan-configuration-api/pull-requests/125) — adds `httpRequestInjections` to `domain_scan_configuration`, threaded through PUT/GET + scan-planner read endpoints |
| **Fix in (scan-planner)**    | PR [#195](https://bitbucket.org/usercentricscode/scan-planner/pull-requests/195) — wires the field from scan-configuration-api response into the `pageScan.httpRequestInjections` field of POST /Scan |
| **What's being tested**     | The injection config (`queryParameters`, `scope`) round-trips: PUT scan-config → GET scan-planner-read → POST scan-api `/Scan` body → scan-tracker navigates to URLs that carry the params      |
| **Surface**                  | Recorded `pageScan.httpRequestInjections` on scan-api's `GET /Scan/{scanId}`, plus visited URLs in the scan result JSON                                                                         |
| **Smoke target URL**         | <https://ag-uc.github.io/se-manual-qa/se-2770/>                                                                                                                                                |

## 2. Preconditions

| # | Precondition                                                                                                              |
| - | ------------------------------------------------------------------------------------------------------------------------- |
| 1 | Dev scan-configuration-api on commit ≥ `df3e82e` (PR #125 merged)                                                         |
| 2 | Dev scan-planner on commit ≥ `f4287f3` (PR #195 merged)                                                                   |
| 3 | Caller can reach both services through IAP — easiest path is impersonating `service-internal-api@usercentrics-playground` |
| 4 | Caller has `api` user credentials for scan-api (Dashlane / `scan-e2e-tests/.env`)                                         |

## 3. Test steps

```
S1. Create a fresh scan configuration on scan-configuration-api.
    POST /v1/scan-configuration
      { "scanConfigurationId": "<unique>", "scanFrequency": "F30P1" }

S2. Upsert a domain entry with httpRequestInjections.
    PUT /v1/scan-configuration/<unique>
      {
        "scanFrequency": "F30P1",
        "domainScanConfigurations": [{
          "domainUrl": "https://ag-uc.github.io/se-manual-qa/se-2770/",
          "maximumPagesToScan": 1,
          "httpRequestInjections": {
            "queryParameters": { "qa_test": "<probe-value>", "utm_source": "scan" },
            "scope": 2
          }
        }]
      }

S3. Verify the round-trip on scan-configuration-api:
      GET /v1/scan-configuration/<unique>                            → echoes injections
      GET /v1/scan-planner/domain/<domainGuid>                       → injections present
      GET /v1/scan-planner/configuration/<unique>/domains            → injections present

S4. Trigger the scan.
    POST /scheduling/scan-now on scan-planner
      { "scanConfigurationId": "<unique>", "domainUrls": ["https://ag-uc.github.io/se-manual-qa/se-2770/"] }

S5. From scan-planner Cloud Run logs, capture the scan-api scanId(s)
    ("Started scan through Scan API - Scan identifier <uuid>").

S6. Confirm the scan-api recorded payload.
    GET /Scan/<scanId> (on api.scanner.dev.usercentrics.cloud, JWT bearer)
    → configurations.pageScan.httpRequestInjections must be
      { queryParameters: { qa_test, utm_source }, scope: "AllPages" }   // scope=2 ↦ AllPages
      // scope=1 maps to "FirstPage"

S7. (Optional, full end-of-chain) Wait for the scan to reach a terminal
    state, fetch GET /Scan/<scanId>/result → follow the `url` link, search
    the scanned URLs for `?qa_test=<probe-value>`. Each navigation
    should carry it.

S8. Clean up: DELETE /v1/scan-configuration/<unique>.
```

## 4. Pass criteria

| # | Assertion                                                                                                                       |
| - | ------------------------------------------------------------------------------------------------------------------------------- |
| 1 | `POST /v1/scan-configuration` returns `201`                                                                                     |
| 2 | `PUT /v1/scan-configuration/<id>` returns `200` and echoes `httpRequestInjections` byte-for-byte                                |
| 3 | All three scan-configuration-api read endpoints (`GET .../{id}`, `GET .../scan-planner/domain/{guid}`, `GET .../configuration/{id}/domains`) include `httpRequestInjections` |
| 4 | `POST /scheduling/scan-now` returns `200`                                                                                       |
| 5 | scan-planner Cloud Run logs contain `Started scan through Scan API - Scan identifier <uuid>` for the test config                |
| 6 | `GET /Scan/<scanId>` on scan-api returns `configurations.pageScan.httpRequestInjections` with the configured params and the correct scope-name |
| 7 | (Optional) Scan result JSON contains at least one visited URL on this origin with the probe query parameter                     |
| 8 | `DELETE /v1/scan-configuration/<id>` returns `204`                                                                              |

## 5. Cleanup

`DELETE /v1/scan-configuration/<unique>` (step S8) removes the test
configuration. Test scans on dev cost very little; nothing else needs to
be cleaned up.
