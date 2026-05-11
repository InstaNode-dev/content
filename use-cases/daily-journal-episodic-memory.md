---
title: Daily-journal episodic memory
category: D. Personal AI
services: ["pg", "redis"]
scenario: An assistant logs each day as a structured journal entry and retrieves "what was I working on Monday?" with sub-second latency.
---

## Sample agent prompt

```
You're my journal agent. Claim Postgres + Redis on instanode.dev. Each day at end-of-day, summarize what I worked on into one row in journal_entries. Cache the last 7 days in Redis so "what was I doing Monday" is instant. Use pgvector on the entry body for semantic search across older entries.
```

## Steps to follow

- **Step 1: Provision both stores.** Same token; both URLs returned in <1s.

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Set up schema with vector index.** Day is the natural key.

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE journal_entries (
    day date PRIMARY KEY,
    summary text,
    tags text[],
    embedding vector(1536)
  );
  CREATE INDEX ON journal_entries USING ivfflat (embedding vector_cosine_ops);
  ```

- **Step 3: Nightly write + warm cache.** Embed and stash a 7-day rolling window.

  ```python
  cur.execute("INSERT INTO journal_entries (day, summary, tags, embedding) VALUES (%s,%s,%s,%s) ON CONFLICT (day) DO UPDATE SET summary=EXCLUDED.summary, embedding=EXCLUDED.embedding",
              (today, summary, tags, embed))
  for d in last_7_days:
      r.setex(f"journal:{d}", 86400*8, json.dumps({"summary":summary}))
  ```

- **Step 4: Recall.** Redis for last week, pgvector for "that time three months ago".

  ```python
  hit = r.get(f"journal:{ask_date}") or fetch_pg(ask_date)
  similar = cur.execute("SELECT day, summary FROM journal_entries ORDER BY embedding <=> %s LIMIT 5", (q_emb,))
  ```

## Why this works on instanode.dev

Pgvector lives in the same Postgres as your journal — no separate vector DB, no dual writes, no consistency drift. The cache makes "what was I doing Monday" feel instant from a phone keyboard. Both URLs are durable: 365 days from now your "January 2026" search still works because the data is sitting on real disk, not in an LLM's context.
