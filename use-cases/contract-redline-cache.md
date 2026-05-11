---
title: Contract redline cache
category: C. Vertical AI apps
services: ["pg", "redis"]
scenario: A contract-review agent caches clause embeddings so re-running redlines on a 200-page MSA is instant on the second pass.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a contract-review agent caches clause embeddings so re-running redlines on a 200-page MSA is instant on the second pass.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're reviewing a 200-page MSA. Claim Postgres + Redis on instanode.dev. Chunk the MSA into clauses, embed each one, store in Postgres with pgvector, and cache clause-hash → redline-output in Redis. On re-run, before calling the LLM for a clause, check Redis first.
```

## Steps to follow

- **Step 1: Provision storage and cache.** Two curls; both return connection URLs in under a second.

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: Set up the clause table.** Hash + embedding indexed for fast similarity lookup.

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE clauses (
    id bigserial PRIMARY KEY,
    clause_hash bytea UNIQUE,
    text text,
    embedding vector(1536)
  );
  ```

- **Step 3: Cache-check before LLM call.** Hash the clause and probe Redis.

  ```python
  h = hashlib.sha256(clause.encode()).hexdigest()
  cached = r.get(f"redline:{h}")
  if cached:
      return json.loads(cached)
  redline = llm.redline(clause)
  r.setex(f"redline:{h}", 86400, json.dumps(redline))
  ```

- **Step 4: Persist embeddings for the next MSA.** Future contracts reuse known-good redlines via similarity.

  ```sql
  INSERT INTO clauses (clause_hash, text, embedding) VALUES ($1, $2, $3)
  ON CONFLICT (clause_hash) DO NOTHING;
  ```

## Why this works on instanode.dev

Same-token provisioning means Postgres and Redis share the same anonymous JWT — no two-service signup, no IAM. Both URLs are AES-256 encrypted at rest. The second pass over the same MSA hits Redis 100% of the time and skips $40 of GPT-4 calls. When the deal closes, claim the token and the resources upgrade to a permanent hobby plan without re-provisioning.
