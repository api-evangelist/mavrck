---
name: mavrck-channel-mix-and-roi
description: Compare Later Influence (Mavrck) programme performance across social networks and compute return on investment by channel, then drill into individual posts.
api: openapi/mavrck-social-network-level-api-openapi.yml
operations:
  - getToken
  - getAllowedInstances
  - getNetworkPerformance
  - findReturnOfInvestments
  - getPostPerformance
generated: '2026-08-12'
method: generated
source: openapi/mavrck-social-network-level-api-openapi.yml, openapi/mavrck-post-level-api-openapi.yml, openapi/mavrck-instance-level-api-openapi.yml, conventions/mavrck-conventions.yml
---

# Channel mix and ROI on Later Influence (Mavrck)

Base URL: `https://api.mavrck.co`. Authenticate first with `getToken` — see
`skills/mavrck-pull-campaign-performance.md` step 1; the token exchange is
identical and the same 12-hour ceiling applies.

## 1. Scope the query — `getAllowedInstances`

`GET /v1/reporting/instances`. Collect the instance ids. Pass several
`instanceIds` values to aggregate across workspaces in one call, or omit the
parameter entirely to span everything the credentials reach.

## 2. Channel mix — `getNetworkPerformance`

`GET /v1/reporting/network/performance`

Returns performance grouped by social network. This is the call that answers
"where is the programme actually working" — Instagram vs TikTok vs YouTube vs
Facebook vs Pinterest.

Same base filters as every other performance endpoint: `startDate`, `endDate`
(both required, range ≤ 2 years), `instanceIds[]`, `campaignIds[]` (max 50),
`reportingGroupIds[]`, `pageSize` (≤ 100), `pageNumber`, `sortProperty`,
`sortDirection`.

## 3. ROI by channel — `findReturnOfInvestments`

`GET /v1/reporting/network/return-on-investment`

The ROI/ROAS breakdown per network. Run it over the *same* `startDate`/`endDate`
window as step 2 — mixing windows between the two calls is the easiest way to
produce a report that does not reconcile.

## 4. Drill into the content — `getPostPerformance`

`GET /v1/reporting/post/performance`

Post-level analytics. Use it to explain a channel result rather than to build
the top-line: this is the highest-cardinality endpoint on the API, and with a
100-row page cap and no rate-limit signal it is the one most likely to make you
hammer the host.

## Reconciliation rules that matter

- **Do not sum post-level rows to reproduce a network total.** Analytics land on the day they are *recorded*, not the day the content was published; a post outside your window can still contribute to an in-window network figure. Use `dateBasis` semantics from the provider's docs when you need to reason about this.
- **`null` ≠ `0`.** `null` means the network does not expose that metric for that row (Facebook Reels, Facebook Live and Facebook profile content are self-reported by the influencer and not API-backed at all). `0` is a real zero.
- **Paid vs organic.** Every paid metric carries the `paid` prefix. A "total impressions" number that mixes `impressions` and `paidImpressions` is your own invention, not the API's.
- Only the networks Later Influence actually integrates return API-backed data: Facebook Graph API, the Instagram API, TikTok One API and the Pinterest Analytics API.

## Failure modes

Both endpoints return `500`/`502` on upstream trouble. When the platform
surface's error envelope carries a `parent` object, the fault is in the social
network, not in Mavrck, and retrying will not help until that network recovers —
see `errors/mavrck-problem-types.yml`.

Check `https://status.later.com/` before escalating; Later Influence incidents
are published there in prose even though no Later Influence *component* exists
on the page.
