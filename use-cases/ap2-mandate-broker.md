---
title: AP2 mandate broker
category: O. Cross-agent commerce & payments
services: ["pg", "webhook"]
scenario: A Google-AP2-style mandate service stores a user's shopping mandate in Postgres, lets buyer agents request approval via webhook, and emits signed receipts for each purchase.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Google-AP2-style mandate service stores a user's shopping mandate in Postgres, lets buyer agents request approval via webhook, and emits signed receipts for each purchase.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're a Google-AP2-style mandate broker. Users register shopping mandates in Postgres. Buyer agents POST to /mandates/{id}/spend with a candidate purchase; you validate against the mandate's constraints, fire a webhook to the merchant if approved, and persist the signed receipt back to the row. Reject if the mandate is expired or over-budget.
```

## Steps to follow

- **Step 1: Provision Postgres + webhook.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  HOOK=$(curl -sX POST https://api.instanode.dev/webhook/new -H "Authorization: Bearer $T" | jq -r .receive_url)
  ```

- **Step 2: Mandate + receipts schema.**

  ```sql
  CREATE TABLE mandates (
    id uuid PRIMARY KEY,
    user_id text NOT NULL,
    budget_cents int NOT NULL,
    spent_cents int NOT NULL DEFAULT 0,
    expires_at timestamptz NOT NULL
  );
  CREATE TABLE receipts (
    receipt_id uuid PRIMARY KEY,
    mandate_id uuid REFERENCES mandates(id),
    amount_cents int,
    merchant text,
    signed_at timestamptz DEFAULT now()
  );
  ```

- **Step 3: Atomic spend — only succeeds if within budget and unexpired.**

  ```sql
  UPDATE mandates
  SET spent_cents = spent_cents + $2
  WHERE id = $1
    AND spent_cents + $2 <= budget_cents
    AND expires_at > now()
  RETURNING id;
  ```

- **Step 4: On success, fire the webhook with a signed receipt.**

  ```python
  receipt = sign_receipt(mandate_id, amount, merchant, BROKER_KEY)
  requests.post(MERCHANT_WEBHOOK, json=receipt)
  pg.execute("INSERT INTO receipts VALUES (%s,%s,%s,%s)",
             (receipt["id"], mandate_id, amount, merchant))
  ```

## Why this works on instanode.dev

Mandate brokers live or die on atomic budget checks — concurrent buyer agents racing to spend can't double-debit. Postgres' conditional UPDATE solves it in one statement. The webhook URL is a stable, replayable endpoint that the merchant can subscribe to, so you don't need Pub/Sub or SNS. Two curls, full AP2-shaped flow.

## Related cases

- [AP2 mandate audit trail](/use-cases/ap2-mandate-audit-trail.md) — the archival half of this live broker
- [Agent-marketplace escrow](/use-cases/agent-marketplace-escrow.md) — complementary commitment layer for buyer-seller flows
- [Stripe-ACP checkout agent](/use-cases/stripe-acp-checkout-agent.md) — the Stripe-rail alternative to the AP2 spec
