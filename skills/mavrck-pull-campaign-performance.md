---
name: mavrck-pull-campaign-performance
description: Authenticate against the Later Influence (Mavrck) Reporting API and pull campaign-level performance — engagements, impressions, estimated ROI, tracking and affiliate conversions — for a date range.
api: openapi/mavrck-campaign-level-api-openapi.yml
operations:
  - getToken
  - getAllowedInstances
  - getCampaignPerformance
generated: '2026-08-12'
method: generated
source: openapi/mavrck-authentication-api-openapi.yml, openapi/mavrck-instance-level-api-openapi.yml, openapi/mavrck-campaign-level-api-openapi.yml, conventions/mavrck-conventions.yml, errors/mavrck-problem-types.yml
---

# Pull campaign performance from Later Influence (Mavrck)

Base URL: `https://api.mavrck.co`

> **Read this first.** This is the **v1** Reporting API and the provider has
> publicly announced it will be deprecated. See `lifecycle/mavrck-lifecycle.yml`
> — there is no firm sunset date and **no `Sunset` header is emitted**, so the
> only warning you will get is the help-centre article. New integrations should
> target the successor.

## 1. Exchange credentials for a JWT — `getToken`

`POST /oauth/token`, `Content-Type: application/json`, body
`{"clientId": "...", "clientSecret": "..."}`. The response carries the token in
a **`jwt`** member (not `access_token`).

Credentials are not self-serve; they are issued by a Later Influence Account
Manager.

Send it on every later call as `Authorization: Bearer <jwt>`.

Lifetime is published inconsistently — 12 hours in the spec, 24 hours in the
help centre. **Treat 12 hours as the ceiling and refresh on any 401.**

Failures return `application/problem+json`:

- `401` `ANL_00401` / "Invalid Client Credentials" — wrong id or secret. Do not retry.
- `CLIENT_DISABLED` — the client account is turned off. Do not retry.
- `NO_ACCESSIBLE_INSTANCES` — the client has no active instance associations. Escalate to the Account Manager; retrying will never fix it.

## 2. Find out what you can query — `getAllowedInstances`

`GET /v1/reporting/instances`

Returns the instance ids your credentials reach. An **instance** is one Later
Influence workspace. Every performance call is scoped by instance, so do this
first rather than assuming an id.

## 3. Pull campaign performance — `getCampaignPerformance`

`GET /v1/reporting/campaign/performance`

Required: `startDate` and `endDate` (`YYYY-MM-DD`). **The range may not exceed
two years.**

Optional: `instanceIds[]` (omit to span every instance you can reach),
`campaignIds[]` (**max 50 items**), `reportingGroupIds[]`, `pageSize` (default
50, **max 100**), `pageNumber` (default 1), `sortProperty`, `sortDirection`
(default `DESC`).

`sortProperty` is validated server-side against a fixed list the contract only
states in prose. The full enum is lifted into
`overlays/mavrck-reporting-api-overlay.yaml` under `x-sort-properties`. Common
choices: `estimatedRoi`, `engagements`, `impressions`, `engagementRate`,
`estimatedValueGenerated`, `affiliateLinksRoi`.

> Watch the spelling: **`affiliateLinksComissionEarned`** is misspelled upstream
> (one `m`). Send it exactly as written or the sort is rejected.

## 4. Page through the result

The envelope is `{ "data": {...}, "pagination": { page, pageSize, totalPages,
totalItems } }`. Loop `pageNumber` until `page == totalPages`.

Do **not** reuse a pagination reader written for the Mavrck platform API — that
surface pages with `limit`/`offset` and wraps results in `{ "meta", "data" }`.
Two incompatible conventions live on this one host; see
`conventions/mavrck-conventions.yml`.

## 5. Reading the numbers

- Data is current only to **the previous day** — several social networks apply a 24-hour reporting delay.
- Analytics are recorded daily and appear only when the recording date falls inside your range. A post published last week whose first interaction lands today shows up only when *today* is in the range.
- Organic metrics use plain names (`impressions`, `engagements`); paid metrics always carry the `paid` prefix (`paidImpressions`, `paidCpm`).
- `null` means the metric is **not available** for that row. `0` means it is defined and the value is zero. Do not coalesce them.

## Error handling

There is **no rate-limit header and no `Retry-After` anywhere on this API** —
see `rate-limits/mavrck-rate-limits.yml`. Back off on your own schedule.

There is also **no request-id header**, so nothing to quote in a support ticket.

Retry `500`/`502` with exponential backoff. Do not retry `400`, `401`, `403` or
`404` — fix the request or the credentials.
