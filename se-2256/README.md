# SE-2256 — scan-adapter DomainOrig GTM-exclusion smoke test plan

## 1. Scope

|                          |                                                                                                                                                                                                                                                                                                          |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ticket**               | [SE-2256](https://usercentrics.atlassian.net/browse/SE-2256) / `CTS-4017`                                                                                                                                                                                                                                |
| **Fix in**               | `scan-adapter` PR [#23427](https://dev.azure.com/cybot/Scanner%20Services/_git/scan-adapter/pullrequest/23427) — merged 2026-06-08, commit `3c03b8d`                                                                                                                                                     |
| **What's being tested**  | `DomainOrigResolver` now excludes `www.googletagmanager.com` (+ non-www variant) from being used as `DomainOrig`. When the recorded tracker `Initiator` host matches the exclusion list, the resolver falls back to `tracker.ProviderDomain`.                                                              |
| **Surface**              | Cookie declaration for a Cookiebot dev test customer pointed at the smoke page (and/or the production declaration for `vaccindirekt.se` after the dev → prod deploy).                                                                                                                                    |
| **Smoke target URL**     | <https://ag-uc.github.io/se-manual-qa/se-2256/>                                                                                                                                                                                                                                                          |
| **Canonical reproducer** | `vaccindirekt.se` (Cookiebot domain ID `1717503`) — the ticket reporter. Production scans before the deploy attribute `tt_appInfo` / `tt_sessionId` / `tt_pixel_session_index` / `lastExternalReferrer*` / `pagead/1p-user-list/#` to `www.googletagmanager.com`; after the deploy they should not. |

## 2. Why this page exists (and what it deliberately does not do)

The fix is a pure host-string match in `DomainOrigResolver.ExtractDomainOrigInformationFromInitiator`:

```csharp
if (InitiatorExclusionHosts.Contains(initiatorUrl.Host))
{
    return new IDomainOrigResult.Ignore();
}
```

Anything the scanner records with `Initiator host == www.googletagmanager.com` (or `googletagmanager.com`) takes the new
branch. So this page only needs to cause the scanner to record at least one cookie/storage entry whose `Initiator` host
is `googletagmanager.com`. That is achieved by loading Google's `gtag.js` from
`www.googletagmanager.com/gtag/js?id=G-DEMO12345` — the bundle still fetches and sets the standard GA cookies, and the
scanner records the `Initiator` URL on each of those sets.

This page **does not** synthesise the original TikTok / Meta / Bing-via-GTM symptom shape. That shape requires a real
GTM container that fires those tags, which is owned per-customer and cannot be reproduced from a static HTML page. The
canonical end-to-end check for that shape is a fresh scan of `vaccindirekt.se` on Cookiebot dev (or comparing pre/post-
deploy production declarations) — see section 4 below.

## 3. Preconditions

| #   | Precondition                                                                                                                                                                                                |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Dev `scan-adapter` is running the post-fix build (PR #23427 merged + redeployed). `git log --all --oneline \| grep 23427` in the `scan-adapter` repo should show `3c03b8d` reachable from the deployed tag. |
| 2   | Cookiebot dev has a test customer + CBID whose start URL is configured to `https://ag-uc.github.io/se-manual-qa/se-2256/` (or any page that loads `gtag.js` with this Initiator pattern).                   |
| 3   | Tester has access to the CB Admin / Manager UI for the test CBID, **or** can render the cookie declaration script in a browser.                                                                             |
| 4   | (For section 4 only) the tester can view production scans for Cookiebot domain id `1717503` (`vaccindirekt.se`) or another already-affected customer.                                                       |

## 4. Test steps

### Path A — synthetic smoke against this page (no third-party container required)

```text
S1. Trigger a fresh scan on the test CBID pointed at
    https://ag-uc.github.io/se-manual-qa/se-2256/

S2. Wait for the scan to reach a terminal state and for scan-adapter
    post-processing to finish (~5 min depending on environment).

S3. Open the cookie declaration for the test CBID, language = English.

S4. Find each entry the scanner picked up (typically GA's _ga / _gid plus
    a few network-request rows). For every row, inspect the "DomainOrig"
    / "Source" field exposed in the dev declaration UI (or query the
    underlying scan-adapter output).

S5. PASS if NO row attributes DomainOrig to www.googletagmanager.com.
    FAIL if any row still attributes DomainOrig to www.googletagmanager.com
    — that would mean the exclusion list isn't hit, or the deploy is
    behind PR #23427.
```

### Path B — canonical real-world reproducer (vaccindirekt.se)

```text
B1. Trigger a fresh Cookiebot scan of vaccindirekt.se on dev for domain
    ID 1717503 (or use the post-deploy production scan once it lands).

B2. Compare the resulting cookie declaration's "Google" vendor group
    against the pre-fix declaration (scans 2c2b840b-…, 9cab44d9-…,
    3974bd9c-… are quoted in the ticket and assess the same shape).

B3. Confirm that these entries — previously grouped under Google — are
    no longer attributed to www.googletagmanager.com:

      tt_appInfo, tt_pixel_session_index, tt_sessionId   (→ analytics.tiktok.com)
      _uetsid, _uetsid_exp, _uetvid, _uetvid_exp, MUID   (→ bing.com)
      lastExternalReferrer, lastExternalReferrerTime    (→ connect.facebook.net)
      pagead/1p-user-list/#                             (→ google.com or the cookie's real provider)

B4. PASS if the above land under TikTok / Microsoft / Meta groups (or
    their respective ProviderDomains) instead of under Google.
    FAIL if any of those entries is still grouped under Google.
```

### Path C — code-level confirmation (no scan required)

This is the cheapest signal that the rule is wired correctly. The PR ships two new theory cases in
`ResolveDomainOrigTests.cs`:

```text
[Theory]
[InlineData("https://www.googletagmanager.com/gtm.js?id=GTM-XXXXX")]
[InlineData("https://googletagmanager.com/gtm.js?id=GTM-XXXXX")]
public void WHEN_Initiator_host_is_a_known_generic_loader_and_provider_domain_is_third_party_THEN_Use_provider_domain_as_domain_orig(string initiator)
{
    // tracker.ProviderDomain = "www.third-party.com"
    // expectedDomainOrig    = "www.third-party.com"
    // → asserts both www-prefixed and bare host go to ProviderDomain
}
```

Run the unit tests from the repo root:

```bash
cd ~/Projects/scan/scan-adapter
git pull --ff-only
dotnet test test/Cookiebot.PostProcessing.ScanAdapter.Core.Adapter.UnitTests \
  --filter "FullyQualifiedName~ResolveDomainOrigTests" --nologo
```

Both new InlineData rows should pass. CI has already exercised this on the merge; running locally is a sanity check, not
an independent signal.

## 5. Pass / fail summary

| Path                         | PASS                                                                                                                                                                                                | FAIL                                                                                                                                            |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| A — synthetic smoke          | No row in the dev declaration attributes `DomainOrig` to `www.googletagmanager.com`.                                                                                                                | At least one row still attributes `DomainOrig` to `www.googletagmanager.com` after the deploy.                                                  |
| B — vaccindirekt.se rescan   | TikTok / Bing / Meta entries no longer appear under the **Google** vendor group; each appears under its real ProviderDomain (`analytics.tiktok.com`, `bing.com`, `connect.facebook.net`).           | Any of those entries still groups under **Google (12)** as in the original ticket screenshot.                                                   |
| C — unit-test re-run         | Both new `WHEN_Initiator_host_is_a_known_generic_loader_…` theory rows pass.                                                                                                                        | Either theory row fails → regression on the InitiatorExclusionHosts wiring.                                                                     |

## 6. Out of scope

- **Tag managers other than GTM.** The PR's exclusion list contains only `www.googletagmanager.com` (and its non-www
  variant). Other intermediaries (Tealium iQ via `tags.tiqcdn.com`, AMP via `cdn.ampproject.org`, OneTrust via
  `cdn.cookielaw.org`, etc.) are still treated as the initiator host and may keep producing misattributed `DomainOrig`
  values in the wild. The ticket comment lists those as future work; this smoke does not cover them.
- **The "both third-party — which one wins" decision** described in the CTS-4017 comment for OneTrust-on-bt.dk
  (`pluto.tv` vs `cdn.cookielaw.org` vs `www.bt.dk`). PR #23427 only fixes the GTM-loader instance of that class. The
  product-side decision on the more nuanced cases lives elsewhere.
- **Multi-language cookie declaration content** (out-of-scope per SE-3091 precedent — separate Cookiebot DB content
  concern).

## 7. Verification surfaces — practical notes

The same surface caveats from SE-3091 apply here:

- The public `consent.cookiebot.com/<cbid>/cd.js` is a loader bundle (~11 KB JS); cookie data is fetched at runtime.
  `curl | grep` against the cd.js will not work.
- Preferred: open the test domain's declaration page in the CB Admin / Manager UI and inspect each row's source.
- Alternative: build a tiny HTML page with
  `<script id="CookieDeclaration" src="https://consent.cookiebot.com/<test-cbid>/cd.js" type="text/javascript" async></script>`
  and open it; the rendered declaration table exposes the resolved vendor grouping.
