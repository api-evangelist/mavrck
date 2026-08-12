---
name: mavrck-receive-workflow-stage-change-webhook
description: Build a receiver for the one webhook Mavrck (Later Influence) emits — the workflow stage change event fired when an influencer joins a campaign or moves between campaign stages.
api: asyncapi/mavrck-webhooks-asyncapi.yml
operations: []
generated: '2026-08-12'
method: generated
source: webhooks/mavrck-workflow-stage-change-event-webhook-openapi.json, webhooks/mavrck-webhooks.yml, asyncapi/mavrck-webhooks-asyncapi.yml
---

# Receive the Mavrck workflow stage change webhook

This is an **event** skill, not a REST call — there are no operationIds to
invoke. Mavrck POSTs to *you*.

Mavrck publishes exactly one outbound webhook contract. Register your HTTPS
receiver in the Later Influence platform UI; there is **no webhook management
API**, so an integration cannot provision or rotate its own subscription
programmatically.

## The payload

```json
{
  "event": {
    "campaign":   { "id": 0, "instanceId": 0 },
    "influencer": { "id": 0, "membershipId": 0 },
    "current":    { "label": "…", "slug": "…" },
    "previous":   { "label": "…", "slug": "…" }
  },
  "subscriberId": "…",
  "token": "…"
}
```

- `previous` is **null** when the influencer was newly *added* to the campaign rather than moved between stages. That is how you tell a join from a transition.
- `current.slug` / `previous.slug` are **immutable** stage ids — key your state machine on `slug`.
- `current.label` / `previous.label` are **mutable** display names. Never key on `label`; a marketer renaming a stage in the UI will silently break you.
- `influencer.membershipId` is nullable. `influencer.id` identifies the person; `membershipId` identifies that person's relationship to *your* instance.
- `subscriberId` varies when one endpoint receives several instances' events, or when several applications receive one instance's events. Use it to route.

## Verifying the sender — read this carefully

Authorization is a **static 16-character alphanumeric `token` carried in the
request body**. You read the expected value from the Mavrck platform and compare
it on receipt.

There is **no HMAC signature header, no timestamp, and no replay protection**.
This is the weakest common webhook authentication pattern, and it has real
consequences for your receiver:

1. Compare `token` with a **constant-time** comparison, not `==`.
2. Terminate TLS yourself and reject plain HTTP — the token is transmitted in cleartext inside the body on every delivery, so the transport is the only confidentiality you get.
3. Treat the payload as a **notification, not a source of truth**. Re-read state from the Reporting API before acting on anything consequential; you cannot detect a replay.
4. Rotating the token requires a change in the platform UI, so plan for a window where both old and new are accepted.

## Responding

Mavrck reads your HTTP status as the delivery result:

| You return | Mavrck does |
|---|---|
| `200` | Treats the event as delivered |
| `401`, `403`, `404`, `405` | **Gives up — no retry** |
| `500`, `502` | Retries |

Backoff schedule and maximum attempt count are **not published**.

The practical rule: if your handler cannot process an event but the event is
valid, return **500**, not 4xx. A 4xx makes Mavrck drop the event permanently,
and there is no replay endpoint to recover it from.

Acknowledge fast and process asynchronously — the provider publishes no
delivery timeout, so do not do the work inside the request.

## What you will not get

No campaign, payment, content-approval or reward events exist. Every other
state change in the platform must be discovered by polling the REST surface.
