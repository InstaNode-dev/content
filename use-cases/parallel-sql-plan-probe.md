---
title: Parallel SQL-plan probe
category: M. Parallel tool execution
services: ["pg", "redis"]
scenario: A data-analyst agent issues 12 candidate EXPLAIN ANALYZE queries against a forked Postgres in parallel, picking the lowest-cost plan to run against the real DB.
---

## Sample agent prompt

```
Probe 12 candidate query rewrites in parallel against a forked Postgres. Each probe runs EXPLAIN ANALYZE, the lowest-cost plan wins, and we cache the chosen rewrite per query-shape hash in Redis so we don't re-probe on the next call. Provision a fresh Postgres and Redis.
```

## Steps to follow

- **Step 1: Provision the fork-target DB and Redis.**

  ```bash
  PG=$(curl -s -X POST https://api.instanode.dev/db/new | jq -r .connection_url)
  RD=$(curl -s -X POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Seed it with a snapshot from production.**

  ```bash
  pg_dump $PROD_URL --schema-only | psql $PG
  pg_dump $PROD_URL --data-only --table=orders --table=users | psql $PG
  ```

- **Step 3: Run 12 EXPLAINs concurrently.**

  ```python
  async def explain(sql):
      async with pool.acquire() as c:
          return await c.fetchval(f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {sql}")
  costs = await asyncio.gather(*[explain(s) for s in candidates])
  best = min(zip(candidates, costs), key=lambda x: x[1][0]["Plan"]["Total Cost"])
  ```

- **Step 4: Cache the winning rewrite.**

  ```python
  await redis.setex(f"plan:{shape_hash(original_sql)}", 3600, best[0])
  ```

- **Step 5: Next call peeks the cache.**

  ```python
  cached = await redis.get(f"plan:{shape_hash(sql)}")
  return cached.decode() if cached else None
  ```

## Why this works on instanode.dev

Spinning up a throwaway Postgres for plan probing used to take an RDS dashboard tour; here it's one curl that returns a connection string in a second. Redis caches plan decisions so a hot query shape is probed once an hour, not once a request.
