---
title: Live agent-topology graph
category: N. Multi-agent observability
services: ["nats", "pg"]
scenario: Each spawn/handoff event is published to a NATS subject; a topology agent subscribes and maintains a real-time graph of which parent spawned which child in Postgres for the dashboard.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: each spawn/handoff event is published to a NATS subject; a topology agent subscribes and maintains a real-time graph of which parent spawned which child in Postgres for the dashboard.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a live topology graph of our agent fleet. Every spawn or handoff publishes an event to NATS with parent_id and child_id. A topology agent subscribes, upserts edges into Postgres, and exposes a query for the React dashboard to render the live tree. Provision NATS and Postgres and return URLs.
```

## Steps to follow

- **Step 1: Provision NATS and Postgres.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"stream":"topo"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: Edge schema.**

  ```sql
  CREATE TABLE agent_edges (
    parent_id text NOT NULL,
    child_id text NOT NULL,
    kind text NOT NULL,
    created_at timestamptz DEFAULT now(),
    PRIMARY KEY (parent_id, child_id)
  );
  CREATE INDEX ON agent_edges (parent_id);
  ```

- **Step 3: Publish spawn/handoff from each agent.**

  ```python
  await nc.publish("topo.event",
      json.dumps({"parent": parent, "child": child, "kind": "spawn"}).encode())
  ```

- **Step 4: Topology subscriber upserts edges.**

  ```python
  async def on_event(msg):
      e = json.loads(msg.data)
      pg.execute("""INSERT INTO agent_edges VALUES (%s,%s,%s)
                    ON CONFLICT DO NOTHING""", e["parent"], e["child"], e["kind"])
  await nc.subscribe("topo.event", cb=on_event)
  ```

- **Step 5: Recursive query for the dashboard tree.**

  ```sql
  WITH RECURSIVE tree AS (
    SELECT parent_id, child_id, 0 AS depth FROM agent_edges WHERE parent_id = $1
    UNION ALL
    SELECT e.parent_id, e.child_id, t.depth+1
    FROM agent_edges e JOIN tree t ON e.parent_id = t.child_id
  ) SELECT * FROM tree;
  ```

## Why this works on instanode.dev

NATS gives the swarm a single event bus that any spawning library can publish to with one line. Postgres' recursive CTEs render the tree without an extra graph DB, and both resources live under one token so the topology service deploys with two curls.

## Related cases

- [Live agent status broadcast](/use-cases/live-agent-status-broadcast.md) — the heartbeat source this topology graph subscribes to
- [Agent-run lineage store](/use-cases/agent-run-lineage-store.md) — persistent-history counterpart to the live spawn graph
- [OpenTelemetry agent-trace ingest](/use-cases/opentelemetry-agent-trace-ingest.md) — OTel-spans variant of the same span/edge ingestion
