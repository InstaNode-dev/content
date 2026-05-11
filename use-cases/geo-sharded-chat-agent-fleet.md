---
title: Geo-sharded chat-agent fleet
category: R. Edge agents & federated swarms
services: ["redis", "pg"]
scenario: Region-pinned agents in EU, US, and APAC each own a local Redis for session state, replicating only consented memories to a central Postgres for cross-region handoff.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: region-pinned agents in EU, US, and APAC each own a local Redis for session state, replicating only consented memories to a central Postgres for cross-region handoff.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a geo-sharded chat fleet. In each region (US/EU/APAC), claim a local Redis on instanode.dev for session state. Claim one shared Postgres for consented cross-region memories. Region pin agents by edge location; only "memorable" messages (user opt-in) replicate to Postgres.
```

## Steps to follow

- **Step 1: Provision per-region Redis.** Use region hint on the request.

  ```bash
  for r in us-east eu-west ap-south; do
    curl -sX POST https://api.instanode.dev/cache/new \
      -H "Content-Type: application/json" -d "{\"region\":\"$r\"}" \
      | jq -r .connection_url > redis-$r.url
  done
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  ```

- **Step 2: Session state writes locally.** Edge agent reads its REDIS_URL env var.

  ```python
  r = redis.from_url(os.environ["REDIS_URL"])
  r.hset(f"session:{user_id}", mapping={"last_msg": msg, "ts": time.time()})
  ```

- **Step 3: Memorable messages replicate.** Gated by user opt-in flag.

  ```python
  if user_opted_in_memory:
      pg.execute("INSERT INTO memories(user_id, region, content, embedding) VALUES (%s,%s,%s,%s)",
                 (user_id, region, msg, embed))
  ```

- **Step 4: Cross-region handoff.** Roaming user reads from central Postgres on new region's Redis miss.

  ```python
  state = r.hgetall(f"session:{user_id}")
  if not state:
      mems = pg.execute("SELECT content FROM memories WHERE user_id=%s ORDER BY ts DESC LIMIT 20", (user_id,))
      r.hset(f"session:{user_id}", "warm", json.dumps(mems))
  ```

## Why this works on instanode.dev

Region-pinned Redis keeps per-session reads on the same continent as the user — chat latency stays under 50ms. The shared Postgres carries only consented, dense memories, so cross-region replication bandwidth is bounded by user choice, not by chat volume. One control plane for all four resources; one token model; no AWS region toggling, no Aurora Global setup.
