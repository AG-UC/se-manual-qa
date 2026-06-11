# SE-3248 — DEA multi-method ExternalReference smoke test plan

## 1. Scope

| | |
| --- | --- |
| **Ticket** | [SE-3248](https://usercentrics.atlassian.net/browse/SE-3248) — Investigate NullReferenceException in GCM check results and empty Completed states |
| **Fix under test** | `domain-evaluation` PR [#76](https://bitbucket.org/usercentricscode/domain-evaluation/pull-requests/76) — merged 2026-06-11 (commit `dee93c0`) |
| **Follow-up** | [SE-3294](https://usercentrics.atlassian.net/browse/SE-3294) — DB-level unique index on `(client, id, method)` (out of scope here) |
| **Smoke target URL** | <https://ag-uc.github.io/se-manual-qa/se-3248/> |
| **What's being tested** | DEA's `ExternalReference → OrderId` association is now method-aware, so the same `(clientIdentifier, externalId)` can be reused across `ConsentModeCheck` and `CbConfigurationCheck` without collision. Polling the **wrong-method** endpoint for an order returns **404**, not the deceptive `Completed/result=null/error=null` payload that caused the NRE in CB Admin. |

## 2. Why this page exists

Unlike SE-2686 (which needed purpose-built trackers), SE-3248 is a pure
**API/storage-layer fix** — the URI being scanned does not matter, only that
DEA finishes the scan quickly enough to test the post-scan lookup. This page
is a trivial static HTML so the `Completed` state arrives within a handful
of seconds and the test is fast.

The fix has three behaviour changes (per PR #76):

1. `RegisterAssociation` now keys on `(client, id, method)` — one association per method.
2. `LookupAssociation` prefers an exact-method match, and **falls back to legacy method-less docs** so existing references keep resolving.
3. Each service's `GetEvaluationResult(orderId)` returns `null` (→ 404) when a Completed order's results carry a different method, instead of `Completed/result=null/error=null`.

## 3. Preconditions

1. DEA dev (commit `dee93c0` or later) is deployed — confirm with
   `gcloud deploy releases list --delivery-pipeline=domain-evaluation-api-pipeline --region=europe-west3 --project=uc-ci-cd | grep dee93c0`
   (rollout state `SUCCEEDED`).
2. You have the dev API key (`DEA_API_KEY_DEV` env var per `scan-bruno` setup; otherwise
   `gcloud secrets versions access latest --secret=domain-evaluation-api-dev-team-api-key-blue --project=uc-domain-evaluation-dev`).
3. (Optional) `scan-bruno` is set up locally — already has the
   `Domain Evaluation API > ConsentModeCheck` and `> CbConfigurationCheck` requests.

## 4. Test recipe

The recipe is two POSTs that share an `externalReference`, then four GETs. Pick a fresh `client`/`id` per run so previous attempts don't contaminate the result — the snippets below use a timestamp suffix.

### Option A — Bruno

```
1. scan-bruno > Domain Evaluation API > On-demand DEV.
2. POST api/v1/ConsentModeCheck — body:
     {
       "uris": ["https://ag-uc.github.io/se-manual-qa/se-3248/"],
       "externalReference": { "clientIdentifier": "se-3248", "externalId": "smoke-<UNIQUE>" }
     }
   Save the returned orderId as GCMOrderId (the post-response script does this).
3. POST api/v1/CbConfigurationCheck — body:
     {
       "uris": ["https://ag-uc.github.io/se-manual-qa/se-3248/"],
       "externalReference": { "clientIdentifier": "se-3248", "externalId": "smoke-<UNIQUE>" }
     }
   Save the returned orderId as CbOrderId (must be DIFFERENT from GCMOrderId).
4. Poll until both complete:
     GET api/v1/ConsentModeCheck?orderId={{GCMOrderId}}        → state: Completed
     GET api/v1/CbConfigurationCheck?orderId={{CbOrderId}}     → state: Completed
5. Lookup by externalReference — each method must resolve to its OWN order:
     GET api/v1/ConsentModeCheck/ByExternalReference?client=se-3248&id=smoke-<UNIQUE>      → orderId == GCMOrderId
     GET api/v1/CbConfigurationCheck/ByExternalReference?client=se-3248&id=smoke-<UNIQUE>  → orderId == CbOrderId
6. Defensive-guard cross-check — polling the WRONG endpoint must 404:
     GET api/v1/ConsentModeCheck?orderId={{CbOrderId}}         → HTTP 404
     GET api/v1/CbConfigurationCheck?orderId={{GCMOrderId}}    → HTTP 404
```

### Option B — Raw curl

```bash
# 0. Credentials + a unique id for this run
export DEA="https://domain-evaluation-api-ondemand.domain-evaluation.dev.usercentrics.cloud"
export API_KEY="<DEA_API_KEY_DEV>"
export ID="smoke-$(date +%s)"
export PAGE="https://ag-uc.github.io/se-manual-qa/se-3248/"

# 1. Register the same externalReference under BOTH methods
GCM_ORDER=$(curl -s -X POST "$DEA/api/v1/ConsentModeCheck" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d "{\"uris\":[\"$PAGE\"],\"externalReference\":{\"clientIdentifier\":\"se-3248\",\"externalId\":\"$ID\"}}" \
  | jq -r .orderId)
CB_ORDER=$(curl -s -X POST "$DEA/api/v1/CbConfigurationCheck" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d "{\"uris\":[\"$PAGE\"],\"externalReference\":{\"clientIdentifier\":\"se-3248\",\"externalId\":\"$ID\"}}" \
  | jq -r .orderId)
echo "GCM=$GCM_ORDER  CB=$CB_ORDER  (must differ)"

# 2. Wait for both Completed
for o in "$GCM_ORDER" "$CB_ORDER"; do
  until curl -s "$DEA/api/v1/ConsentModeCheck?orderId=$o" -H "x-api-key: $API_KEY" \
        | jq -e '.state == "Completed"' >/dev/null \
     || curl -s "$DEA/api/v1/CbConfigurationCheck?orderId=$o" -H "x-api-key: $API_KEY" \
        | jq -e '.state == "Completed"' >/dev/null
  do sleep 3; done
done

# 3. ByExternalReference must return each method's OWN order
curl -s "$DEA/api/v1/ConsentModeCheck/ByExternalReference?client=se-3248&id=$ID" \
  -H "x-api-key: $API_KEY" | jq '{orderId, state}'   # orderId == GCM_ORDER
curl -s "$DEA/api/v1/CbConfigurationCheck/ByExternalReference?client=se-3248&id=$ID" \
  -H "x-api-key: $API_KEY" | jq '{orderId, state}'   # orderId == CB_ORDER

# 4. Wrong-method GET ?orderId= must 404 (NOT Completed/result=null/error=null)
curl -s -o /dev/null -w "ConsentModeCheck on Cb order:   %{http_code}\n" \
  "$DEA/api/v1/ConsentModeCheck?orderId=$CB_ORDER" -H "x-api-key: $API_KEY"
curl -s -o /dev/null -w "CbConfigurationCheck on GCM order: %{http_code}\n" \
  "$DEA/api/v1/CbConfigurationCheck?orderId=$GCM_ORDER" -H "x-api-key: $API_KEY"
```

## 5. Pass criteria

| # | Check | Expected | Why it matters |
| - | ----- | -------- | -------------- |
| 1 | Two POSTs with the same externalReference return **different** orderIds | yes | proves the association is no longer overwritten across methods (PR #76 Part 1) |
| 2 | `ConsentModeCheck/ByExternalReference` returns the GCM orderId | yes | proves the lookup picks the right method's order |
| 3 | `CbConfigurationCheck/ByExternalReference` returns the CB orderId | yes | symmetric proof |
| 4 | `GET /ConsentModeCheck?orderId={CbOrderId}` returns **HTTP 404** | yes (and **NOT** `Completed/result=null/error=null`) | proves the defensive guard (PR #76 Part 2) kills the NRE-causing payload |
| 5 | `GET /CbConfigurationCheck?orderId={GcmOrderId}` returns **HTTP 404** | yes | symmetric proof |

## 6. Backward-compatibility note (legacy method-less associations)

The PR includes a fallback in `LookupAssociation` that picks up legacy
Mongo documents written before this deploy (where the `Method` field is
absent). The fallback is unit-tested in
`Usercentrics.DomainEvaluationApi.Application.Tests`. We don't synthesize
a legacy doc in this smoke recipe — confirmation that "we don't break legacy
orders" is observed in DEV/PROD by:

- **Datadog signal:** the query *"Error while getting consent mode check result"*
  used in SE-3248 should trend toward zero post-deploy.
- **Service log signal:** the new per-service warning `"Order {OrderId} holds {Method} results, not …"`
  fires only on stale cross-method polls — its volume should also decay as data self-heals.
