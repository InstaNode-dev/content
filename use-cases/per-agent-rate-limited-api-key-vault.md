---
title: Per-agent rate-limited API key vault
category: O. Cross-agent commerce & payments
services: ["redis", "pg"]
scenario: A vault service mints scoped, rate-limited API keys for child agents on demand; Redis tracks per-key usage and the spend rolls up to a Postgres billing event stream.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a vault service mints scoped, rate-limited API keys for child agents on demand; Redis tracks per-key usage and the spend rolls up to a Postgres billing event stream.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a vault that mints scoped, rate-limited API keys for child agents on demand. Redis tracks per-key usage with sliding windows, and every charge rolls up to a Postgres billing event stream. Provision both and expose mint() and charge() endpoints.
```

## Steps to follow

- **Step 1: Provision Redis and Postgres.**

  ```bash
  curl -s -X POST https://api.instanode.dev/cache/new
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: Billing event schema.**

  ```sql
  CREATE TABLE billing_events (
    id bigserial PRIMARY KEY,
    parent_agent text NOT NULL,
    child_key text NOT NULL,
    units numeric NOT NULL,
    reason text,
    at timestamptz DEFAULT now()
  );
  CREATE INDEX ON billing_events (parent_agent, at DESC);
  ```

- **Step 3: Mint a scoped key.**

  ```python
  def mint(parent, scopes, rpm):
      key = "vk_" + secrets.token_urlsafe(24)
      redis.hset(f"key:{key}", mapping={"parent": parent, "scopes": ",".join(scopes), "rpm": rpm})
      return key
  ```

- **Step 4: Sliding-window check on each use.**

  ```python
  def allow(key):
      now = time.time()
      pipe = redis.pipeline()
      pipe.zremrangebyscore(f"rl:{key}", 0, now - 60)
      pipe.zadd(f"rl:{key}", {str(uuid4()): now})
      pipe.zcard(f"rl:{key}")
      pipe.expire(f"rl:{key}", 70)
      _, _, count, _ = pipe.execute()
      return count <= int(redis.hget(f"key:{key}", "rpm"))
  ```

- **Step 5: Charge on each call.**

  ```sql
  INSERT INTO billing_events (parent_agent, child_key, units, reason)
  VALUES ($1, $2, $3, $4);
  ```

## Why this works on instanode.dev

Redis' sorted-set sliding windows are the canonical agent-rate-limit primitive, and Postgres keeps the immutable charge trail your finance system reconciles against. Two curls, no separate gateway service.
