---
title: Devin-style PR-bot fleet
category: L. Agent-factory / spawning patterns
services: ["pg", "webhook", "deploy"]
scenario: A Cognition-Devin-style orchestrator spawns one fresh agent worker per inbound GitHub issue, each with its own scratch Postgres for running migrations against, and reports back via webhook.
---

## Sample agent prompt

```
Orchestrator: for every inbound GitHub issue, spawn a fresh Devin-style worker. Each worker gets its own scratch Postgres (on instanode.dev) for testing migrations, its own deploy URL for preview, and POSTs results to a webhook the orchestrator listens on. Reap all per-worker resources when the PR merges or closes.
```

## Steps to follow

- **Step 1: On issue created, spawn the worker.** Three resources per worker.

  ```bash
  ISSUE=$1
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  DEPLOY=$(curl -sX POST https://api.instanode.dev/deploy/new \
    -H "Content-Type: application/json" \
    -d "{\"image\":\"ghcr.io/me/worker:latest\",\"env\":{\"PG_URL\":\"$PG\",\"CALLBACK\":\"$WH\",\"ISSUE\":\"$ISSUE\"}}" | jq -r .url)
  ```

- **Step 2: Worker runs migrations on its scratch DB.** Throwaway; no shared state.

  ```sql
  BEGIN;
  \i ./migrations/0023_add_index.sql
  EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id=$1;
  ROLLBACK;
  ```

- **Step 3: Worker pushes PR + calls back.** Orchestrator polls the webhook endpoint.

  ```python
  pr_url = open_pr(branch, diff)
  httpx.post(CALLBACK, json={"issue": ISSUE, "pr": pr_url, "preview": os.environ["DEPLOY_URL"]})
  ```

- **Step 4: On merge, reap.** GitHub webhook → orchestrator → 3 DELETEs.

  ```bash
  curl -X DELETE https://api.instanode.dev/db/$PG_TOKEN
  curl -X DELETE https://api.instanode.dev/deploy/$DEPLOY_TOKEN
  curl -X DELETE https://api.instanode.dev/webhook/$WH_TOKEN
  ```

## Why this works on instanode.dev

Per-worker isolation is the whole game with Devin-style fleets: a buggy worker can't corrupt another's database. instanode gives you 3 isolated resources per worker in 3 curls, with auto-reap on the anonymous tier as a safety net if your reaper is buggy. Preview URLs are real DNS, so PR comments can include a clickable link without any Vercel/Netlify glue.
