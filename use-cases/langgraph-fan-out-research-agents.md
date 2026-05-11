---
title: LangGraph fan-out research agents
category: J. Agent swarms & fan-out
services: ["nats", "pg"]
scenario: A planner agent splits a research question into 12 sub-queries and dispatches them to parallel LangGraph workers that each hit different sources, returning JSON to a NATS subject the reducer subscribes to.
---

## Sample agent prompt

```
Implement a LangGraph planner that decomposes a research question into 12 sub-queries and dispatches them to parallel workers over NATS. Each worker writes its JSON result to a results subject and persists raw findings in Postgres. The reducer subscribes, waits for all 12, and synthesizes a final answer. Provision the queue and DB and return the URLs.
```

## Steps to follow

- **Step 1: Provision NATS and Postgres.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"stream":"research"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: Schema for findings.**

  ```sql
  CREATE TABLE findings (
    run_id uuid NOT NULL,
    sub_query text NOT NULL,
    result jsonb NOT NULL,
    PRIMARY KEY (run_id, sub_query)
  );
  ```

- **Step 3: Planner node publishes 12 sub-queries.**

  ```python
  for sq in plan.sub_queries:
      await js.publish(f"research.work.{run_id}", json.dumps({"q": sq, "run": run_id}).encode())
  ```

- **Step 4: Worker subscribes, calls source APIs, persists to Postgres.**

  ```python
  async def worker(msg):
      data = await fetch_sources(msg["q"])
      pg.execute("INSERT INTO findings VALUES (%s,%s,%s)", run_id, msg["q"], data)
      await js.publish(f"research.done.{run_id}", json.dumps(data).encode())
  ```

- **Step 5: Reducer waits for 12 messages on the done subject and synthesizes.**

  ```python
  sub = await nc.subscribe(f"research.done.{run_id}")
  collected = [await sub.next_msg() for _ in range(12)]
  return await llm.synthesize(collected)
  ```

## Why this works on instanode.dev

The same token gives the planner, the workers, and the reducer a shared queue and store with no broker config. JetStream's per-subject filtering lets the reducer key its wait on `run_id`, so concurrent research runs do not block each other.
