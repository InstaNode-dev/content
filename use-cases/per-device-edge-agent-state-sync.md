---
title: Per-device edge-agent state sync
category: R. Edge agents & federated swarms
services: ["webhook", "pg"]
scenario: An IoT fleet of 10,000 edge agents each runs on a Workers-style runtime, syncing local SQLite deltas to a central Postgres via webhook batches every minute.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an IoT fleet of 10,000 edge agents each runs on a Workers-style runtime, syncing local SQLite deltas to a central Postgres via webhook batches every minute.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
We've got 10,000 Workers-runtime edge agents each with a local SQLite. Every minute they batch their deltas and POST to a webhook receiver. A central worker drains the receiver, applies upserts to Postgres, and tracks last-sync per device. Provision webhook and Postgres.
```

## Steps to follow

- **Step 1: Provision the receiver and central DB.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: Central schema.**

  ```sql
  CREATE TABLE device_state (
    device_id text NOT NULL,
    key text NOT NULL,
    value jsonb,
    version bigint,
    PRIMARY KEY (device_id, key)
  );
  CREATE TABLE sync_cursor (device_id text PRIMARY KEY, last_at timestamptz);
  ```

- **Step 3: Edge agent batches once a minute.**

  ```javascript
  setInterval(async () => {
    const deltas = await sqlite.all("SELECT * FROM outbox WHERE synced = 0")
    if (!deltas.length) return
    await fetch(env.WH_URL, { method: "POST", body: JSON.stringify({ device: env.ID, deltas }) })
    await sqlite.exec("UPDATE outbox SET synced = 1")
  }, 60_000)
  ```

- **Step 4: Central drain worker.**

  ```python
  for msg in pull_webhook_batch(WH_URL, n=500):
      with pg.transaction():
          for d in msg["deltas"]:
              pg.execute("""INSERT INTO device_state(device_id,key,value,version) VALUES (%s,%s,%s,%s)
                            ON CONFLICT (device_id,key) DO UPDATE SET value=EXCLUDED.value, version=EXCLUDED.version
                            WHERE device_state.version < EXCLUDED.version""",
                         msg["device"], d["k"], d["v"], d["ver"])
          pg.execute("INSERT INTO sync_cursor VALUES (%s, now()) ON CONFLICT (device_id) DO UPDATE SET last_at=now()", msg["device"])
  ```

- **Step 5: Detect stragglers.**

  ```sql
  SELECT device_id, last_at FROM sync_cursor WHERE last_at < now() - interval '5 minutes';
  ```

## Why this works on instanode.dev

The webhook receiver flattens 10k devices into a single buffered ingestion point, so the drain worker controls Postgres concurrency. Version-aware upserts make the sync idempotent across retries.

## Related cases

- [Geo-sharded chat-agent fleet](/use-cases/geo-sharded-chat-agent-fleet.md) — region-scoped variant of the same edge-to-central sync idea
- [Federated-learning aggregator](/use-cases/federated-learning-aggregator.md) — edge-fleet sibling that syncs gradients instead of SQLite deltas
- [OpenTelemetry agent-trace ingest](/use-cases/opentelemetry-agent-trace-ingest.md) — similar webhook-batch ingestion at agent-fleet scale
