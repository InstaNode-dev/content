---
title: Stripe-ACP checkout agent
category: O. Cross-agent commerce & payments
services: ["webhook", "mongo"]
scenario: A shopping agent uses the Stripe + OpenAI Agentic Commerce Protocol to place an order; the merchant agent receives a webhook with the cart, persists it in Mongo, and ships.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a shopping agent uses the Stripe + OpenAI Agentic Commerce Protocol to place an order; the merchant agent receives a webhook with the cart, persists it in Mongo, and ships.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Implement the merchant side of Stripe + OpenAI's Agentic Commerce Protocol. Provision a webhook via instanode.dev to receive the ACP "create_checkout" event, persist the cart into MongoDB, and reply with a checkout_id. On payment_succeeded, ship the order and append the fulfillment status to the same Mongo document.
```

## Steps to follow

- **Step 1: Provision webhook + Mongo.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new)
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  echo "Register $(echo $WH | jq -r .receive_url) with Stripe as ACP endpoint"
  ```

- **Step 2: Drain ACP events.**

  ```python
  events = requests.get(f"{webhook_base}/{token}/requests?since={cursor}").json()
  for e in events:
      body = e["body"]
      if body["type"] == "checkout.create":
          db.carts.insert_one({
              "_id": body["data"]["checkout_id"],
              "cart": body["data"]["line_items"],
              "status": "pending"
          })
  ```

- **Step 3: Reply on the Stripe HTTP callback** with the created checkout_id and a payment intent.

- **Step 4: On payment_succeeded, fulfill.**

  ```python
  db.carts.update_one(
      {"_id": checkout_id},
      {"$set": {"status": "shipped", "tracking": tracking_number, "fulfilled_at": now()}}
  )
  ```

- **Step 5: Stream order status** back to the shopping agent via the ACP `order.update` event.

## Why this works on instanode.dev

The webhook resource captures every ACP event with a queryable history (requests endpoint), so the merchant agent never silently drops an order even on a crash. MongoDB's document model fits the variable shape of cart line items without a normalized schema.

## Related cases

- [Agent-marketplace escrow](/use-cases/agent-marketplace-escrow.md) — escrow-rail alternative to direct ACP checkout
- [AP2 mandate broker](/use-cases/ap2-mandate-broker.md) — AP2-spec equivalent to ACP for agent commerce
- [Stripe-event entitlements](/use-cases/stripe-event-entitlements.md) — downstream entitlement update after ACP fires its webhook
