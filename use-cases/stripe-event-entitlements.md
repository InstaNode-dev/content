---
title: Stripe-event entitlements
category: H. Indie & SaaS founders
services: ["webhook", "redis", "pg"]
scenario: A SaaS receives Stripe webhooks, updates user plan tier, and invalidates a Redis entitlement cache atomically.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a SaaS receives Stripe webhooks, updates user plan tier, and invalidates a Redis entitlement cache atomically.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Redis + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
On every Stripe webhook (customer.subscription.created, updated, deleted), update the user's plan_tier in Postgres and invalidate the Redis entitlement cache for that user atomically. Provision webhook + Postgres + Redis via instanode.dev. Use a single transaction in Postgres and a Redis DEL keyed by user_id.
```

## Steps to follow

- **Step 1: Provision all three.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"stripe-event-entitlements-webhook"}' | jq -r .receive_url)
  DB=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"stripe-event-entitlements-db"}' | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"stripe-event-entitlements-cache"}' | jq -r .connection_url)
  ```

- **Step 2: Schema.**

  ```sql
  CREATE TABLE users (
    id TEXT PRIMARY KEY,
    plan_tier TEXT NOT NULL DEFAULT 'free',
    updated_at TIMESTAMPTZ
  );
  ```

- **Step 3: Stripe handler.**

  ```python
  def handle(evt):
      stripe.Webhook.construct_event(evt.body, evt.headers["Stripe-Signature"], WEBHOOK_SECRET)
      with conn.transaction():
          conn.execute("UPDATE users SET plan_tier=%s, updated_at=now() WHERE id=%s",
                       (tier_from_price(evt.price), evt.user_id))
      r.delete(f"ent:{evt.user_id}")
  ```

- **Step 4: App reads entitlements with cache-aside.**

  ```python
  tier = r.get(f"ent:{uid}") or load_and_cache(uid)
  ```

- **Step 5: Fail-open on Redis** — if cache is down the app re-reads from Postgres and serves.

## Why this works on instanode.dev

The webhook resource gives you Stripe-signature-verified replay history out of the box, so a brief downtime won't lose entitlement changes — drain on recovery. Postgres + Redis on one platform means one network hop and one auth model for both.

## Related cases

- [Pre-launch waitlist store](/use-cases/pre-launch-waitlist-store) — the pre-paid signup half of the same indie-founder funnel
- [Solo-founder analytics warehouse](/use-cases/solo-founder-analytics-warehouse) — downstream warehouse that consumes plan-tier events
- [Stripe-ACP checkout agent](/use-cases/stripe-acp-checkout-agent) — upstream agent-checkout flow that fires these webhooks
