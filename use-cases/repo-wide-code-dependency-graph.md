---
title: Repo-wide code dependency graph
category: A. AI coding agents
services: ["pg"]
scenario: An agent indexes a polyrepo into a blast-radius graph so it can answer "what breaks if I rename this function" before editing.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an agent indexes a polyrepo into a blast-radius graph so it can answer "what breaks if I rename this function" before editing.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Index every Go, Python, and TypeScript file in this monorepo into a blast-radius graph. Provision a Postgres via instanode.dev, create symbols/edges tables, walk the repo with tree-sitter, and insert one row per function definition plus one row per call edge. Then answer "what breaks if I rename pkg/auth.VerifyJWT".
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"repo-wide-code-dependency-graph-db"}' | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Create the graph schema.**

  ```sql
  CREATE TABLE symbols (
    id BIGSERIAL PRIMARY KEY,
    repo TEXT, path TEXT, name TEXT, kind TEXT,
    line INT, UNIQUE(repo, path, name, line)
  );
  CREATE TABLE edges (
    src BIGINT REFERENCES symbols(id),
    dst BIGINT REFERENCES symbols(id),
    kind TEXT
  );
  CREATE INDEX ON edges (dst);
  ```

- **Step 3: Walk the repo with tree-sitter and bulk-insert.**

  ```python
  for fn in walk_definitions(repo):
      sid = insert_symbol(fn)
      for callee in fn.calls:
          insert_edge(sid, resolve(callee), "calls")
  ```

- **Step 4: Blast-radius query.**

  ```sql
  WITH RECURSIVE callers AS (
    SELECT src FROM edges WHERE dst = (SELECT id FROM symbols WHERE name='VerifyJWT')
    UNION SELECT e.src FROM edges e JOIN callers c ON e.dst = c.src
  ) SELECT path, name FROM symbols WHERE id IN (SELECT src FROM callers);
  ```

- **Step 5: Feed the result back into the agent's edit-planning step** before it touches the rename.

## Why this works on instanode.dev

Postgres recursive CTEs make blast-radius traversal a single query, and provisioning takes one HTTP call so an agent can rebuild the index on every branch without managing a long-lived service. The 5GB pro tier holds millions of edges for a large monorepo.

## Related cases

- [Coding-agent cross-session memory](/use-cases/coding-agent-cross-session-memory) — complementary architectural-memory store for the same coding agent
- [Multi-repo shared scratchpad](/use-cases/multi-repo-shared-scratchpad) — cross-repo coordination view layered on top of the graph
- [SARIF scan-result store](/use-cases/sarif-scan-result-store) — uses similar Postgres patterns for cross-commit graph drift
