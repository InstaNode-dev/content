---
title: Claude Code agent-teams scratchpad
category: J. Agent swarms & fan-out
services: ["pg", "redis"]
scenario: A lead Claude Code session spawns 6 sibling worker sessions, each owning one module of a monorepo; they coordinate by reading and writing a shared scratchpad table.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a lead Claude Code session spawns 6 sibling worker sessions, each owning one module of a monorepo; they coordinate by reading and writing a shared scratchpad table.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're the lead Claude Code session orchestrating 6 worker sessions, each owning one module of a monorepo (auth, billing, search, web, mobile, infra). They coordinate by writing status updates to a shared scratchpad table in Postgres and a "currently editing" lock in Redis. Before touching a file, a worker SET-NX-locks it.
```

## Steps to follow

- **Step 1: Provision the coordination plane.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"claude-code-agent-teams-scratchpad-db"}'   -H "Authorization: Bearer $T" | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"claude-code-agent-teams-scratchpad-cache"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
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

## Related cases

- [Multi-repo shared scratchpad](/use-cases/multi-repo-shared-scratchpad) — the Mongo-document version of the same shared-coordination idea
- [Coding-agent cross-session memory](/use-cases/coding-agent-cross-session-memory) — longer-horizon memory that survives across sessions, not just teams
- [Shared episodic memory store](/use-cases/shared-episodic-memory-store) — generalizes the scratchpad table beyond coding agents
