---
title: x402 micropayment per tool call
category: O. Cross-agent commerce & payments
services: ["redis", "pg", "webhook"]
scenario: An agent calling another agent's premium MCP tool receives HTTP 402, settles a 0.3-cent USDC payment, and retries; the receiver tracks paid calls in Redis with a Postgres ledger.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an agent calling another agent's premium MCP tool receives HTTP 402, settles a 0.3-cent USDC payment, and retries; the receiver tracks paid calls in Redis with a Postgres ledger.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + Postgres + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Receiver side of x402: provision Redis + Postgres + webhook via instanode.dev. The MCP tool returns HTTP 402 on first call. After the client settles 0.3 cents in USDC and retries with the tx receipt, verify via webhook from the chain indexer, INCR a Redis counter "paid:<caller>:<tool>", and persist the ledger row to Postgres. Tool only executes once paid:* is positive.
```

## Steps to follow

- **Step 1: Provision all three.**

  ```bash
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  DB=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  ```

- **Step 2: Return 402 on unpaid call.**

  ```python
  if r.decr(f"paid:{caller}:{tool}") < 0:
      r.incr(f"paid:{caller}:{tool}")
      return Response(402, headers={"X-Payment-Required": "USDC 0.003 to 0xMERCHANT"})
  ```

- **Step 3: Chain indexer fires webhook on confirmed USDC transfer.**

  ```python
  for evt in drain_webhook():
      caller, tool = parse_memo(evt["body"])
      r.incr(f"paid:{caller}:{tool}")
      conn.execute("INSERT INTO ledger(caller,tool,tx_hash,amount_cents,paid_at) VALUES (%s,%s,%s,%s,now())",
                   (caller, tool, evt["body"]["tx"], 30))
  ```

- **Step 4: Client retries with the tx receipt header.** The DECR now succeeds and the tool runs.

- **Step 5: Reconciliation query** shows revenue per tool over time.

  ```sql
  SELECT tool, date_trunc('hour', paid_at), sum(amount_cents) FROM ledger GROUP BY 1,2;
  ```

## Why this works on instanode.dev

Redis `DECR`/`INCR` is atomic so concurrent calls can't double-spend a single payment. The webhook gives the chain indexer a verified-payload sink that the receiver can drain on its own schedule, and Postgres makes the per-tool revenue trail queryable from day one.

## Related cases

- [x402 micropayment ledger](/use-cases/x402-micropayment-ledger.md) — the principal-balance reconciliation layer above per-call x402
- [Per-agent rate-limited API key vault](/use-cases/per-agent-rate-limited-api-key-vault.md) — rate-limited alternative to payment-gated tool access
- [Agent-marketplace escrow](/use-cases/agent-marketplace-escrow.md) — job-level commitment layer above per-call settlement
