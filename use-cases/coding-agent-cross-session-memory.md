---
title: Coding-agent cross-session memory
category: A. AI coding agents
services: ["pg"]
scenario: A terminal-resident coding agent persists architectural decisions across days so it can recall "what we tried Tuesday" via pgvector similarity search.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a terminal-resident coding agent persists architectural decisions across days so it can recall "what we tried Tuesday" via pgvector similarity search.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're my terminal coding agent. Claim a Postgres on instanode.dev for our long-term decision log, enable pgvector, and create a decisions table with an embedding column. On every architectural choice we make, write a row with the decision text + an embedding. When I ask "what did we try on X", run a cosine similarity search and surface the top 3 rows.
```

## Steps to follow

- **Step 1: Claim a Postgres.** One unauthenticated POST returns a live connection URL; no signup needed for a 24h trial.

  ```bash
  curl -sX POST https://api.instanode.dev/db/new | tee pg.json
  export PG_URL=$(jq -r .connection_url pg.json)
  ```

- **Step 2: Enable pgvector and create the schema.** instanode ships pgvector pre-installed on shared Postgres.

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE decisions (
    id bigserial PRIMARY KEY,
    decided_at timestamptz DEFAULT now(),
    topic text,
    body text,
    embedding vector(1536)
  );
  CREATE INDEX ON decisions USING ivfflat (embedding vector_cosine_ops);
  ```

- **Step 3: Log a decision at the end of each session.** The agent calls OpenAI embeddings, then inserts.

  ```python
  emb = openai.embeddings.create(input=body, model="text-embedding-3-small").data[0].embedding
  cur.execute("INSERT INTO decisions(topic, body, embedding) VALUES (%s,%s,%s)", (topic, body, emb))
  ```

- **Step 4: Recall "what did we try Tuesday".** Embed the query, cosine search.

  ```sql
  SELECT topic, body, decided_at FROM decisions
  ORDER BY embedding <=> $1::vector LIMIT 3;
  ```

## Why this works on instanode.dev

Pgvector is enabled out of the box on every provisioned Postgres, so there's no extension install dance. The connection URL is a real, persistent database — not a sandbox that resets — and survives across terminal sessions, IDE restarts, and machine reboots. Claim it once, and the same JWT lets the agent recover the URL on any new device.

## Related cases

- [Claude Code agent-teams scratchpad](/use-cases/claude-code-agent-teams-scratchpad.md) — the across-siblings counterpart to across-sessions memory
- [Multi-repo shared scratchpad](/use-cases/multi-repo-shared-scratchpad.md) — scratchpad that spans repos instead of time
- [Obsidian-vault embedding sync](/use-cases/obsidian-vault-embedding-sync.md) — same pgvector recall pattern over personal notes
