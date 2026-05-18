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
You're my terminal coding agent across a multi-week refactor of our billing service. Last Tuesday we tried wrapping Stripe webhooks in an idempotency table; on Thursday we tried a Redis lock; today I want to compare and pick one. Claim a Postgres on instanode.dev for our long-term decision log (or recover it if a JWT is already in ~/.config/instanode/token), enable pgvector, ensure a `decisions` table exists with topic + body + embedding + tags, and at the end of every session write a row for each major architectural choice we made. When I ask "what did we try about webhooks?" embed the query with text-embedding-3-small and return the top 3 cosine-nearest decisions with their dates.
```

## Steps to follow

We'll thread one running log — the multi-week refactor of a billing service — through every step. By the last command you'll watch the agent recall a decision made nine days ago by similarity, not by keyword.

- **Step 1: Claim a Postgres once, cache the JWT, recover it on every subsequent run.** Cross-session means the URL has to survive `Ctrl-D`. The agent caches the claimed token in `~/.config/instanode/token` so the next session resumes the same database.

  ```bash
  TOKEN_FILE=~/.config/instanode/token
  if [ -f "$TOKEN_FILE" ]; then
    PG_URL=$(curl -s -H "Authorization: Bearer $(cat $TOKEN_FILE)" \
             https://api.instanode.dev/api/v1/resources | jq -r '.[] | select(.type=="postgres") | .connection_url')
  else
    mkdir -p ~/.config/instanode
    RESP=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"coding-agent-cross-session-memory-db"}')
    echo "$RESP" | jq -r .token > "$TOKEN_FILE"
    PG_URL=$(echo "$RESP" | jq -r .connection_url)
    # then visit the printed claim_url within 24h to upgrade from anonymous → hobby
    echo "claim within 24h: $(echo $RESP | jq -r .claim_url)"
  fi
  export PG_URL
  ```

- **Step 2: Enable pgvector and create the schema once.** instanode ships pgvector pre-installed on every provisioned Postgres — no `CREATE EXTENSION` failure on permissions.

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;

  CREATE TABLE IF NOT EXISTS decisions (
    id          bigserial PRIMARY KEY,
    decided_at  timestamptz NOT NULL DEFAULT now(),
    session_id  text NOT NULL,
    topic       text NOT NULL,
    body        text NOT NULL,
    tags        text[] NOT NULL DEFAULT '{}',
    embedding   vector(1536) NOT NULL
  );

  CREATE INDEX IF NOT EXISTS decisions_topic_idx
    ON decisions USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
  ```

- **Step 3: At session end, the agent logs each architectural choice.** Embed → insert. Same code path every time, no manual journaling.

  ```python
  # agent/log_decision.py
  import os, openai, psycopg
  oa = openai.OpenAI()

  def log(session_id: str, topic: str, body: str, tags: list[str]) -> int:
      emb = oa.embeddings.create(
          model="text-embedding-3-small",
          input=f"{topic}\n\n{body}",
          dimensions=1536,
      ).data[0].embedding
      with psycopg.connect(os.environ["PG_URL"]) as cn:
          row = cn.execute(
              "INSERT INTO decisions (session_id, topic, body, tags, embedding) "
              "VALUES (%s, %s, %s, %s, %s) RETURNING id",
              (session_id, topic, body, tags, emb),
          ).fetchone()
          return row[0]

  # called by the agent at the end of Tuesday's session:
  log("2026-04-21", "Stripe webhook idempotency",
      "Tried a Postgres idempotency table keyed on stripe_event_id. "
      "Insert ON CONFLICT DO NOTHING; if rowcount=0, short-circuit. "
      "Concern: table growth — needs a TTL job.",
      ["stripe", "idempotency", "webhooks"])
  ```

- **Step 4: Recall by meaning, not keyword.** When you ask "what did we try about webhooks?" the agent embeds the question and runs a cosine search. Note that "webhook" never appears verbatim in Thursday's Redis-lock body — but the embedding still pulls it.

  ```python
  def recall(question: str, k: int = 3):
      qe = oa.embeddings.create(
          model="text-embedding-3-small", input=question, dimensions=1536
      ).data[0].embedding
      with psycopg.connect(os.environ["PG_URL"]) as cn:
          return cn.execute(
              "SELECT decided_at::date, topic, body, "
              "       1 - (embedding <=> %s::vector) AS similarity "
              "FROM decisions "
              "ORDER BY embedding <=> %s::vector LIMIT %s",
              (qe, qe, k),
          ).fetchall()

  for d, topic, body, sim in recall("what did we try about webhooks?"):
      print(f"[{d}] ({sim:.3f}) {topic}\n  {body[:120]}...\n")
  ```

- **Step 5: Verify recall works across sessions.** Open a fresh terminal — no warm cache, no in-process state — and run the recall.

  ```bash
  # fresh shell, no agent state:
  python -c "from agent.log_decision import recall; \
             [print(r[0], '|', r[1], '|', round(r[3],3)) for r in recall('webhook deduplication', 3)]"
  # 2026-04-21 | Stripe webhook idempotency | 0.872
  # 2026-04-23 | Redis SETNX lock for inbound webhooks | 0.841
  # 2026-04-15 | Outbound retry backoff | 0.612
  ```

## Why this works on instanode.dev

Long-running agent memory has been awkwardly served by two opposite camps. The **Pinecone / Weaviate / Qdrant Cloud** path is a real vector DB but assumes a signup, a project, an API key, and a paid tier the moment you cross the free quota — and worse, an agent that wants to *spawn* its own memory store can't, because Pinecone's project creation isn't an open POST. The **LangChain InMemoryVectorStore** / **Chroma-on-disk** path solves provisioning by being local, but the data dies when the laptop sleeps or when a sibling agent on another machine wants to read it. **OpenAI Assistants memory** locks you into one provider's storage. instanode gives the agent a real Postgres with pgvector pre-installed in one unauthenticated POST — no extension-install dance (a common pgvector failure mode on shared Postgres providers like Supabase free), no project quota, no IAM. The claimed token survives terminal restarts, IDE reloads, machine reboots, and any other device the agent runs on, because the JWT itself is the recovery key. The same `decisions` table is queryable from a Python script, a `psql` shell, or another agent — because it's just Postgres.

## Related cases

- [Claude Code agent-teams scratchpad](/use-cases/claude-code-agent-teams-scratchpad.md) — the across-siblings counterpart to across-sessions memory
- [Multi-repo shared scratchpad](/use-cases/multi-repo-shared-scratchpad.md) — scratchpad that spans repos instead of time
- [Obsidian-vault embedding sync](/use-cases/obsidian-vault-embedding-sync.md) — same pgvector recall pattern over personal notes
