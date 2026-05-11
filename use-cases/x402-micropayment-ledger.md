---
title: x402 micropayment ledger
category: G. Internet-of-AI
services: ["pg"]
scenario: An agent-payment hub records x402 micropayments between agents and reconciles balances per principal.
---

## Sample agent prompt

```
Build an x402 micropayment hub. Provision Postgres via instanode.dev. Every time agent A pays agent B via the x402 HTTP 402 protocol, both sides POST a settlement record (payer, payee, amount_usdc, tx_hash, call_id). The hub records both, double-entry style, and a reconciliation job reports balances per principal hourly.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Double-entry ledger schema.**

  ```sql
  CREATE TABLE entries (
    id BIGSERIAL PRIMARY KEY,
    principal TEXT, counterparty TEXT,
    amount_micros BIGINT,
    direction TEXT CHECK (direction IN ('debit','credit')),
    tx_hash TEXT, call_id TEXT,
    UNIQUE (principal, tx_hash, direction)
  );
  CREATE INDEX ON entries (principal, id);
  ```

- **Step 3: On x402 settlement, write two rows in one transaction.**

  ```python
  with conn.transaction():
      conn.execute("INSERT INTO entries(principal,counterparty,amount_micros,direction,tx_hash,call_id) VALUES (%s,%s,%s,'debit',%s,%s)",
                   (payer, payee, amt, tx, call))
      conn.execute("INSERT INTO entries(principal,counterparty,amount_micros,direction,tx_hash,call_id) VALUES (%s,%s,%s,'credit',%s,%s)",
                   (payee, payer, amt, tx, call))
  ```

- **Step 4: Balance per principal.**

  ```sql
  SELECT principal,
    sum(CASE WHEN direction='credit' THEN amount_micros ELSE -amount_micros END) bal_micros
  FROM entries GROUP BY principal;
  ```

- **Step 5: Hourly reconciliation** flags principals whose balance disagrees with on-chain USDC transfers.

## Why this works on instanode.dev

Postgres transactions guarantee debit + credit land atomically; the UNIQUE constraint makes settlement idempotent under retry. One curl and the agent-payment economy has a real ledger — no Stripe, no custodian.
