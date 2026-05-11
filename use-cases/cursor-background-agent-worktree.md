---
title: Cursor background-agent worktree
category: K. Ephemeral agent runtimes
services: ["redis", "webhook", "deploy"]
scenario: A Cursor background agent claims an isolated dev environment for each branch it's working on, with its own Redis for build caches and a webhook to ping the IDE when the PR is ready.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Cursor background agent claims an isolated dev environment for each branch it's working on, with its own Redis for build caches and a webhook to ping the IDE when the PR is ready.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + webhook receiver + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
For each branch the background agent works on, claim a Redis cache, a webhook receiver, and an instanode deployment slot. Build the branch in the deploy, cache npm/turbo results in Redis, and POST to the webhook when CI passes so the Cursor IDE can pop a toast.
```

## Steps to follow

- **Step 1: Per-branch infra provisioning.** Three curls, all keyed by the same token.

  ```bash
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  DEPLOY=$(curl -sX POST https://api.instanode.dev/deploy/new \
    -H "Content-Type: application/json" \
    -d "{\"image\":\"ghcr.io/me/app:$BRANCH\",\"env\":{\"REDIS_URL\":\"$REDIS\"}}" | jq -r .url)
  ```

- **Step 2: Wire build cache.** turbo + Redis remote cache via the connection URL.

  ```bash
  export TURBO_REMOTE_CACHE_URL=$REDIS
  pnpm turbo run build test
  ```

- **Step 3: Notify IDE on green.** Single POST closes the loop.

  ```bash
  if pnpm turbo run test; then
    curl -X POST $WH -d "{\"branch\":\"$BRANCH\",\"status\":\"green\",\"url\":\"$DEPLOY\"}"
  fi
  ```

- **Step 4: Cursor reads webhook events.** Poll the public requests endpoint.

  ```javascript
  const evts = await fetch(`https://api.instanode.dev/api/v1/webhooks/${token}/requests`).then(r => r.json());
  evts.filter(e => e.body.status === 'green').forEach(toast);
  ```

## Why this works on instanode.dev

Per-branch isolation is what makes background-agent worktrees safe; instanode's per-token resource model means each branch's cache, deploy, and webhook URL are first-class citizens with no IAM ceremony. The deploy URL is a real `*.instanode.dev` subdomain — Cursor can open the preview directly. Reap on branch delete with one DELETE per resource.
