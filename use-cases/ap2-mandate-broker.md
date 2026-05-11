---
title: AP2 mandate broker
category: O. Cross-agent commerce & payments
services: ["pg", "webhook"]
scenario: A Google-AP2-style mandate service stores a user's shopping mandate in Postgres, lets buyer agents request approval via webhook, and emits signed receipts for each purchase.
---

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
