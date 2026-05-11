---
title: AML transaction monitor
category: C. Vertical AI apps
services: ["pg", "nats"]
scenario: A finance agent ingests transaction streams, flags suspicious patterns, and persists compliance decisions with full reasoning trace.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a finance agent ingests transaction streams, flags suspicious patterns, and persists compliance decisions with full reasoning trace.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're an AML compliance agent. Consume transaction events from NATS, score them against your fraud rules, and persist a decision row in Postgres (txn_id, score, flags, reasoning_trace) for every event. Flag any score > 0.8 for human review and emit a downstream NATS event.
```

## Steps to follow

- **Step 1: Provision NATS + Postgres.**

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  PG=$(curl -sX POST https://api.instanode.dev/db/new   -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Decisions table with full audit trail.**

  ```sql
  CREATE TABLE decisions (
    txn_id text PRIMARY KEY,
    score real NOT NULL,
    flags text[],
    reasoning jsonb NOT NULL,
    decided_at timestamptz DEFAULT now(),
    reviewer text
  );
  CREATE INDEX idx_high_score ON decisions (score DESC) WHERE score > 0.8;
  ```

- **Step 3: NATS subscriber, score, persist.**

  ```python
  async for msg in nc.subscribe("transactions.>"):
      txn = json.loads(msg.data)
      score, flags, trace = rules.evaluate(txn)
      pg.execute("""INSERT INTO decisions (txn_id, score, flags, reasoning)
                    VALUES (%s,%s,%s,%s)""",
                 (txn["id"], score, flags, Json(trace)))
      if score > 0.8:
          await nc.publish("aml.review", msg.data)
  ```

- **Step 4: Compliance officer pulls the daily review queue.**

  ```sql
  SELECT txn_id, score, flags, reasoning->>'top_signal' AS signal
  FROM decisions
  WHERE decided_at > now() - interval '24 hours'
    AND reviewer IS NULL AND score > 0.8
  ORDER BY score DESC;
  ```

## Why this works on instanode.dev

AML demands two things: durable audit trail (Postgres ACID) and back-pressure-tolerant event stream (NATS JetStream). The decision row carries its full reasoning JSON so a regulator's "explain this flag" query is one SELECT. Both services are real (not mocks), which matters because compliance teams will literally subpoena your event log.
