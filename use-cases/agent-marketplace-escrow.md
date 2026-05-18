---
title: Agent-marketplace escrow
category: O. Cross-agent commerce & payments
services: ["pg", "webhook"]
scenario: A platform lets agents post jobs and other agents bid; funds sit in an escrow row in Postgres and release via webhook once a verifier agent signs off on delivery.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a platform lets agents post jobs and other agents bid; funds sit in an escrow row in Postgres and release via webhook once a verifier agent signs off on delivery.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're operating a job marketplace where buyer agents post tasks and seller agents bid. When a buyer accepts a bid, lock funds in a Postgres escrow row. The verifier agent calls back via webhook with a signed delivery receipt; on receipt verification, release funds atomically.
```

## Steps to follow

- **Step 1: Provision Postgres + webhook.**

  ```bash
  DB=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"agent-marketplace-escrow-db"}' -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  HOOK=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"agent-marketplace-escrow-webhook"}' -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .receive_url)
  ```

- **Step 2: Escrow schema with state machine.**

  ```sql
  CREATE TYPE escrow_state AS ENUM ('locked','released','refunded','disputed');
  CREATE TABLE escrow (
    job_id uuid PRIMARY KEY,
    buyer_agent text NOT NULL,
    seller_agent text NOT NULL,
    amount_cents int NOT NULL,
    state escrow_state NOT NULL DEFAULT 'locked',
    verifier_sig text,
    updated_at timestamptz DEFAULT now()
  );
  ```

- **Step 3: Lock funds when a bid is accepted.**

  ```sql
  INSERT INTO escrow (job_id, buyer_agent, seller_agent, amount_cents)
  VALUES ($1, $2, $3, $4);
  ```

- **Step 4: Verifier webhook means atomic release.** Only releases if currently `locked`.

  ```sql
  UPDATE escrow
  SET state = 'released', verifier_sig = $2, updated_at = now()
  WHERE job_id = $1 AND state = 'locked'
  RETURNING amount_cents;
  ```

- **Step 5: Subscribe verifier callbacks via the webhook URL.** Configure the verifier service to POST signed receipts to `$HOOK`; your handler reads them and runs Step 4.

## Why this works on instanode.dev

Escrow requires atomic state transitions — exactly one of release/refund/dispute must win. Postgres' `UPDATE ... WHERE state='locked' RETURNING` gives you that for free; no Lua scripts, no distributed lock. The webhook URL is stable and replayable, so a flaky verifier can re-fire without double-releasing. Both resources spin up in under two seconds — fast enough to prototype the marketplace logic before deciding on KYC/regulated rails.

## Related cases

- [AP2 mandate broker](/use-cases/ap2-mandate-broker) — the user-mandate side of the same agent-commerce loop
- [x402 micropayment ledger](/use-cases/x402-micropayment-ledger) — the per-call settlement layer that complements job-level escrow
- [Stripe-ACP checkout agent](/use-cases/stripe-acp-checkout-agent) — fiat-rail equivalent for agent-to-merchant purchases
