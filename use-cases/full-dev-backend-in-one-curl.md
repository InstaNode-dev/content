---
title: Full dev backend in one curl
category: F. Developer tooling
services: ["pg", "redis", "mongo"]
scenario: An AI agent or developer provisions Postgres + Redis + MongoDB anonymously to develop against — no Docker, no cloud account, no installer — and tears it down when done.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an AI agent or developer provisions Postgres + Redis + MongoDB anonymously to develop against — no Docker, no cloud account, no installer — and tears it down when done.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis + MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Give me a full dev backend in one go. Claim Postgres, Redis, and MongoDB on instanode.dev anonymously. Export connection URLs as env vars. Run my app against them. When I'm done, DELETE all three.
```

## Steps to follow

- **Step 1: Three curls in parallel.** Anonymous, no signup, all return in under a second.

  ```bash
  export PG_URL=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  export REDIS_URL=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  export MONGO_URL=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  ```

- **Step 2: Run your app.** Reads env vars; no other setup.

  ```bash
  npm run dev
  ```

- **Step 3: Verify all three are reachable.** Sanity ping from the shell.

  ```bash
  psql $PG_URL -c 'SELECT 1'
  redis-cli -u $REDIS_URL PING
  mongosh "$MONGO_URL" --eval 'db.runCommand({ping:1})'
  ```

- **Step 4: Tear down when done.** Or let the 24h TTL auto-reap.

  ```bash
  for svc in db cache nosql; do
    curl -X DELETE https://api.instanode.dev/$svc/$TOKEN
  done
  ```

## Why this works on instanode.dev

This is the canonical instanode flow: no Docker daemon, no `docker compose up`, no port collisions, no localhost binds. The same URLs work from your laptop, a teammate's laptop, a CI runner, or a deployed container — because they're public endpoints with token auth. Three real production-grade services in three seconds, zero setup, zero account.
