---
title: Per-agent dead-letter inspection queue
category: N. Multi-agent observability
services: ["nats", "mongo"]
scenario: Every failed tool call from a swarm is republished to a NATS DLQ; an investigator agent pulls them in batches, classifies the failure mode, and persists clusters in Mongo for the operator UI.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: every failed tool call from a swarm is republished to a NATS DLQ; an investigator agent pulls them in batches, classifies the failure mode, and persists clusters in Mongo for the operator UI.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Every failed tool call across the swarm publishes to a NATS DLQ subject. Build an investigator agent that consumes the DLQ in batches of 50, classifies the failure (rate-limit, auth, timeout, schema-mismatch), and writes clusters to Mongo for the operator UI. Provision NATS and Mongo.
```

## Steps to follow

- **Step 1: Provision queue and store.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"name":"per-agent-dead-letter-inspection-q-queue","stream":"dlq"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/nosql/new -H 'Content-Type: application/json' -d '{"name":"per-agent-dead-letter-inspection-q-mongo"}'
  ```

- **Step 2: Workers publish failures to the DLQ.**

  ```python
  except Exception as e:
      await js.publish("dlq.tool", json.dumps({
          "agent": agent_id, "tool": tool, "args": args, "err": str(e),
          "stack": traceback.format_exc(), "at": time.time()
      }).encode())
  ```

- **Step 3: Investigator pulls in batches.**

  ```python
  sub = await js.pull_subscribe("dlq.tool", durable="inspector")
  while True:
      batch = await sub.fetch(50, timeout=30)
      classify_and_cluster([json.loads(m.data) for m in batch])
      for m in batch: await m.ack()
  ```

- **Step 4: Cluster docs in Mongo.**

  ```python
  m["clusters"].update_one(
      {"signature": sig},
      {"$inc": {"count": 1}, "$push": {"examples": {"$each": [example], "$slice": -20}},
       "$set": {"last_seen": datetime.utcnow()}},
      upsert=True
  )
  ```

- **Step 5: Operator UI query.**

  ```javascript
  db.clusters.find().sort({count: -1}).limit(20)
  ```

## Why this works on instanode.dev

JetStream's durable pull-consumers let the investigator fall behind during a failure storm without losing messages. Mongo's update-with-upsert collapses thousands of failures into a top-N cluster list with one query.

## Related cases

- [Durable agent task queue](/use-cases/durable-agent-task-queue) — the main queue whose failures feed this DLQ
- [Live agent status broadcast](/use-cases/live-agent-status-broadcast) — complementary live-success-side view of the same swarm
- [Agent-resilience chaos lab](/use-cases/agent-resilience-chaos-lab) — produces the failures the DLQ inspector ends up clustering
