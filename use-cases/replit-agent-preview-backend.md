---
title: Replit-Agent preview backend
category: K. Ephemeral agent runtimes
services: ["pg", "deploy"]
scenario: A Replit-Agent-style preview deploys a one-shot app with a 24h-TTL Postgres so a reviewer can poke at the prototype, then everything is collected when the share link expires.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Replit-Agent-style preview deploys a one-shot app with a 24h-TTL Postgres so a reviewer can poke at the prototype, then everything is collected when the share link expires.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a 24-hour preview backend for this prototype. Provision an anonymous Postgres via instanode.dev (auto-expires in 24h), apply migrations from db/schema.sql, then deploy the FastAPI app from this repo to a public subdomain with DATABASE_URL injected. Print the share URL when done.
```

## Steps to follow

- **Step 1: Provision a 24h-TTL Postgres.** Anonymous tier expires automatically — no cleanup script needed.

  ```bash
  curl -X POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"replit-agent-preview-backend-db"}' | tee db.json
  export DATABASE_URL=$(jq -r .connection_url db.json)
  export DB_TOKEN=$(jq -r .token db.json)
  ```

- **Step 2: Apply schema.**

  ```bash
  psql "$DATABASE_URL" < db/schema.sql
  ```

- **Step 3: Deploy the FastAPI container.** `/deploy/new` accepts a Dockerfile build context and injects env vars.

  ```bash
  curl -X POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $INSTANODE_TOKEN" \
    -F "name=prototype-pr-42" \
    -F "image=ghcr.io/me/prototype:pr-42" \
    -F "env.DATABASE_URL=$DATABASE_URL" | tee deploy.json
  export APP_URL=$(jq -r .url deploy.json)
  ```

- **Step 4: Hand the reviewer one URL.**

  ```bash
  echo "Share: $APP_URL (expires in 24h)"
  ```

- **Step 5: Cleanup is automatic.** The DB's 24h TTL on the anonymous tier reaps the resource; the deploy expires alongside it.

## Why this works on instanode.dev

Anonymous-tier provisions have a 24-hour TTL baked in — exactly the lifecycle a preview backend needs. The token returned from `/db/new` is the same identity used by `/deploy/new`, so the same reviewer link covers both resources and they vanish together without orphaned infrastructure.

## Related cases

- [Sandbox-per-PR preview deployment](/use-cases/sandbox-per-pr-preview-deployment) — PR-scoped variant of the same disposable-preview pattern
- [E2B microVM sandbox per agent turn](/use-cases/e2b-microvm-sandbox-per-agent-turn) — per-turn alternative when previews don't need a share link
- [Side-project container deploy](/use-cases/side-project-container-deploy) — the bare deploy primitive this preview backend wraps
