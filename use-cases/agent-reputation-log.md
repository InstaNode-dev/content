---
title: Agent reputation log
category: G. Internet-of-AI
services: ["pg", "redis"]
scenario: Buyer agents leave ratings on seller agents after each tool call; reputation scores aggregate hourly.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: buyer agents leave ratings on seller agents after each tool call; reputation scores aggregate hourly.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You operate a reputation ledger. Buyer agents POST ratings (1-5) on seller agents after each tool call. Persist every rating in Postgres for auditability; maintain a hot rolling-mean score per seller in Redis for fast lookups. Recompute the Redis cache hourly from Postgres ground truth.
```

## Steps to follow

- **Step 1: Provision both stores.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  ```

- **Step 2: Append-only ratings table.**

  ```sql
  CREATE TABLE ratings (
    id bigserial PRIMARY KEY,
    seller_id text NOT NULL,
    buyer_id text NOT NULL,
    score smallint CHECK (score BETWEEN 1 AND 5),
    rated_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_ratings_seller_time ON ratings (seller_id, rated_at DESC);
  ```

- **Step 3: Write-through to Postgres, increment a Redis hash.**

  ```python
  pg.execute("INSERT INTO ratings (seller_id, buyer_id, score) VALUES (%s,%s,%s)",
             (seller, buyer, score))
  r.hincrby(f"rep:{seller}", "sum", score)
  r.hincrby(f"rep:{seller}", "count", 1)
  ```

- **Step 4: Reputation read is one HMGET, ~1ms.**

  ```python
  s, c = r.hmget(f"rep:{seller}", "sum", "count")
  mean = int(s) / int(c) if c else None
  ```

- **Step 5: Hourly reconciliation job from Postgres.**

  ```sql
  SELECT seller_id, AVG(score)::real, COUNT(*) FROM ratings
  WHERE rated_at > now() - interval '30 days'
  GROUP BY seller_id;
  ```

## Why this works on instanode.dev

Reputation has two access patterns — append-only audit log (Postgres) and read-heavy aggregate (Redis) — and trying to do both in one store kills you. Two curls, two real services, both encrypted at rest. Hourly recon from Postgres means a Redis flush doesn't lose any rating, just rebuilds the cache.

## Related cases

- [A2A agent-card registry](/use-cases/a2a-agent-card-registry.md) — the directory those ratings annotate
- [Token-cost ledger per sub-agent](/use-cases/token-cost-ledger-per-sub-agent.md) — another rollup pattern, but on spend instead of reputation
- [x402 micropayment ledger](/use-cases/x402-micropayment-ledger.md) — reputation tends to gate access to higher-trust paid endpoints
