---
title: Obsidian-vault embedding sync
category: D. Personal AI
services: ["pg"]
scenario: A personal research agent embeds Obsidian notes nightly into pgvector and answers questions across years of writing.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a personal research agent embeds Obsidian notes nightly into pgvector and answers questions across years of writing.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up nightly Obsidian-vault embedding into pgvector on instanode. Provision Postgres, enable pgvector, walk `~/notes/`, chunk by heading, embed with text-embedding-3-small, upsert by path+hash. Expose a search() function I can call from my personal agent.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new | jq -r .connection_url > .pg
  ```

- **Step 2: Enable pgvector and create the schema.**

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE chunks (
    path text NOT NULL, heading text, hash text NOT NULL,
    body text, embedding vector(1536),
    PRIMARY KEY (path, hash)
  );
  CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
  ```

- **Step 3: Nightly sync script chunks by heading and upserts.**

  ```python
  for md in Path("~/notes").expanduser().rglob("*.md"):
      for h, body in chunk_by_heading(md.read_text()):
          h_hash = hashlib.sha256(body.encode()).hexdigest()[:16]
          emb = openai.embeddings.create(model="text-embedding-3-small", input=body).data[0].embedding
          pg.execute("""INSERT INTO chunks VALUES (%s,%s,%s,%s,%s)
                        ON CONFLICT (path, hash) DO NOTHING""", str(md), h, h_hash, body, emb)
  ```

- **Step 4: cron entry.**

  ```bash
  0 3 * * * cd ~/code/notes-embed && uv run sync.py >> sync.log 2>&1
  ```

- **Step 5: search() the agent calls.**

  ```python
  def search(q, k=8):
      e = embed(q)
      return pg.fetch("SELECT path, heading, body FROM chunks ORDER BY embedding <=> %s LIMIT %s", e, k)
  ```

## Why this works on instanode.dev

pgvector ships in the Postgres you provision, so there is no separate vector store to operate. HNSW handles years of notes at low latency, and the same DB powers both your personal agent's RAG and any future search UI.

## Related cases

- [Coding-agent cross-session memory](/use-cases/coding-agent-cross-session-memory.md) — same pgvector recall pattern over code instead of notes
- [Daily-journal episodic memory](/use-cases/daily-journal-episodic-memory.md) — short-form personal counterpart to the long-form vault
- [arXiv-and-RSS research feed](/use-cases/arxiv-and-rss-research-feed.md) — an upstream ingestion source that feeds into the vault
