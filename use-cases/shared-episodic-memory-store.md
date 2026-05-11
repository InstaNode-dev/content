---
title: Shared episodic memory store
category: B. Multi-agent systems
services: ["pg"]
scenario: A planner and a researcher agent read and write the same episodic memory table so each agent sees the other's findings.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a planner and a researcher agent read and write the same episodic memory table so each agent sees the other's findings.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up a shared episodic memory between planner and researcher agents. Provision Postgres via instanode.dev with an episodes table (agent, role, content, embedding, ts). After each tool call either agent writes a row; before each step both read the top-k semantically similar episodes via pgvector cosine search.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -X POST https://api.instanode.dev/db/new | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  ```

- **Step 2: Schema with pgvector.**

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE episodes (
    id BIGSERIAL PRIMARY KEY,
    agent TEXT, role TEXT, content TEXT,
    embedding vector(1536), ts TIMESTAMPTZ DEFAULT now()
  );
  CREATE INDEX ON episodes USING hnsw (embedding vector_cosine_ops);
  ```

- **Step 3: Write after each step.**

  ```python
  emb = openai.embeddings.create(model="text-embedding-3-small", input=content).data[0].embedding
  conn.execute("INSERT INTO episodes(agent,role,content,embedding) VALUES (%s,%s,%s,%s)",
               (agent_id, role, content, emb))
  ```

- **Step 4: Retrieve before each step.**

  ```sql
  SELECT agent, content FROM episodes
  ORDER BY embedding <=> $1 LIMIT 8;
  ```

- **Step 5: Inject into the prompt** as a "what we know so far" block.

## Why this works on instanode.dev

pgvector is preinstalled on instanode Postgres, so two agents sharing memory needs zero extra infra beyond one curl. The hobby tier's 5 connection limit comfortably handles a planner + researcher + one reflection job; pro gives 20 for larger crews.
