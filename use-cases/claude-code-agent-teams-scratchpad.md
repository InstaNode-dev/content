---
title: Claude Code agent-teams scratchpad
category: J. Agent swarms & fan-out
services: ["pg", "redis"]
scenario: A lead Claude Code session spawns 6 sibling worker sessions, each owning one module of a monorepo; they coordinate by reading and writing a shared scratchpad table.
---

## Sample agent prompt

```
You're the lead Claude Code session orchestrating 6 worker sessions, each owning one module of a monorepo (auth, billing, search, web, mobile, infra). They coordinate by writing status updates to a shared scratchpad table in Postgres and a "currently editing" lock in Redis. Before touching a file, a worker SET-NX-locks it.
```

## Steps to follow

- **Step 1: Provision the coordination plane.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new   -H "Authorization: Bearer $T" | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Scratchpad table — every session appends, all read.**

  ```sql
  CREATE TABLE scratchpad (
    id bigserial PRIMARY KEY,
    session text NOT NULL,
    module text NOT NULL,
    note text NOT NULL,
    posted_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_recent ON scratchpad (posted_at DESC);
  ```

- **Step 3: File lock before editing — Redis SET NX EX.**

  ```python
  def claim_file(path, session, ttl=600):
      ok = r.set(f"lock:{path}", session, nx=True, ex=ttl)
      return bool(ok)

  if not claim_file("api/internal/handlers/billing.go", "worker-3"):
      raise RuntimeError("file held by another session")
  ```

- **Step 4: Post a status update so the lead can survey progress.**

  ```sql
  INSERT INTO scratchpad (session, module, note)
  VALUES ('worker-3', 'billing', 'refactored razorpay webhook handler, tests green');
  ```

- **Step 5: Lead's dashboard query.**

  ```sql
  SELECT DISTINCT ON (session) session, module, note, posted_at
  FROM scratchpad ORDER BY session, posted_at DESC;
  ```

## Why this works on instanode.dev

Multi-agent Claude Code sessions need cheap shared state without standing up a service. Postgres for ordered durable notes, Redis for sub-second file locks — the two primitives that solve "who's editing what" cleanly. Two curls, both real, both visible from any of the 6 worker shells via psql/redis-cli for live debugging.
