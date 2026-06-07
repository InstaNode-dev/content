---
title: Tiers and limits
order: 8
---

| Tier       | Postgres    | Redis     | MongoDB      | TTL  | Price       |
| ---------- | ----------- | --------- | ------------ | ---- | ----------- |
| Anonymous  | 10MB / 2c   | 5MB       | 5MB / 2c     | 24h  | free        |
| Hobby      | 1GB / 8c    | 50MB      | 100MB / 5c   | none | $9 / mo     |
| Pro        | 10GB / 20c  | 512MB     | 5GB / 20c    | none | $49 / mo    |
| Team       | unlimited   | unlimited | unlimited    | none | $199 / mo   |

"c" = simultaneous connections. The full table is at `/pricing`.

Hobby Plus and Growth exist in `plans.yaml` as upsell-only intermediate tiers
reached via in-dashboard prompts when a Hobby user hits a quota wall. They are
deliberately omitted from the public tier ladder to keep the customer-facing
comparison simple.

**Team tier status:** Team is live and self-serve at $199/mo (the API no longer
returns `tier_unavailable` for `plan=team`). Note: self-serve checkout for ALL
paid tiers (Hobby/Pro/Team) currently depends on the Razorpay recurring-billing
rollout — until that operator step completes, `POST /api/v1/billing/checkout`
may return a `502`/`503`; contact contact@instanode.dev for assisted onboarding
in the meantime.

Limits are enforced at the Postgres user level (`CONNECTION LIMIT` on the
role) and via per-bucket storage quotas. Exceeding a limit returns a 402 with
an upgrade URL — your app keeps running, the next provision just fails.

## `POST /api/v1/billing/checkout` — concurrent-call dedup

`POST /api/v1/billing/checkout` is server-side deduplicated per team. A second
concurrent call for the same team — within a 60s window — gets a structured
409 instead of a second Razorpay subscription. This catches cross-tab clicks,
mobile double-taps, retried form submits, and agents that retry the endpoint
without coordination.

Response envelope:

```json
{
  "ok": false,
  "error": "checkout_in_flight",
  "message": "A checkout is already being created for this team. Wait ~60s and retry, or visit /dashboard to find the existing pending subscription.",
  "retry_after_seconds": 60,
  "agent_action": "Tell the user a checkout is already being created. They should wait ~60 seconds and refresh — the existing checkout link will appear in the dashboard.",
  "request_id": "..."
}
```

The `retry_after_seconds` field tells callers how long to wait. The TTL also
caps the worst case where the first caller crashes mid-flight — after 60s a
retry is allowed automatically. The standard `Idempotency-Key` header (see
`/docs/idempotency`) is honoured on this route too and provides a longer-window
guarantee — pass it on every retry of a logical checkout attempt.

If Redis is unavailable the dedup guard fails open (the call proceeds), with
a `WARN billing.checkout.dedup_setnx_failed_open` log line. A Redis brownout
must never block a paid upgrade — the idempotency middleware is the second
layer of defence.
