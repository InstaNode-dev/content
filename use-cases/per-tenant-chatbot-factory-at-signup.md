---
title: Per-tenant chatbot factory at signup
category: L. Agent-factory / spawning patterns
services: ["mongo", "redis", "deploy"]
scenario: When a B2B SaaS user clicks 'Add AI assistant', the factory provisions a dedicated Mongo for that tenant's chat history, a Redis for session state, and deploys an isolated agent container.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: when a B2B SaaS user clicks 'Add AI assistant', the factory provisions a dedicated Mongo for that tenant's chat history, a Redis for session state, and deploys an isolated agent container.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB + Redis + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
When a B2B tenant clicks "Add AI assistant," call /nosql/new for their chat history, /cache/new for session state, and /deploy/new for an isolated agent container with those URLs in env. Return the assistant URL to embed. Provision-on-signup, fully tenant-isolated.
```

## Steps to follow

- **Step 1: Tenant clicks the button — your factory backend runs:**

  ```bash
  MONGO=$(curl -s -X POST https://api.instanode.dev/nosql/new -H 'Content-Type: application/json' -d '{"name":"per-tenant-chatbot-factory-at-sign-mongo"}' | jq -r .connection_url)
  REDIS=$(curl -s -X POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"per-tenant-chatbot-factory-at-sign-cache"}' | jq -r .connection_url)
  ```

- **Step 2: Deploy the agent container with tenant-scoped env.**

  ```bash
  curl -s -X POST https://api.instanode.dev/deploy/new \
    -H "Authorization: Bearer $INSTANODE_TOKEN" \
    -F "name=assistant-$TENANT" \
    -F "image=ghcr.io/acme/assistant:latest" \
    -F "env.MONGO_URL=$MONGO" \
    -F "env.REDIS_URL=$REDIS" \
    -F "env.TENANT_ID=$TENANT" \
    | jq -r .url > assistant_url.txt
  ```

- **Step 3: Persist the mapping in the SaaS DB.**

  ```sql
  INSERT INTO tenant_assistants (tenant_id, mongo_url, redis_url, deploy_url)
  VALUES ($1, $2, $3, $4);
  ```

- **Step 4: Embed snippet returned to the tenant's app.**

  ```html
  <script src="https://your-app.com/embed.js" data-assistant="https://assistant-x.instanode.dev"></script>
  ```

- **Step 5: Chat handler in the deployed agent.**

  ```python
  await mongo.history.insert_one({"tenant": TENANT_ID, "role": "user", "text": msg})
  state = await redis.get(f"session:{user_id}")
  ```

## Why this works on instanode.dev

Three resources per tenant, claimed under your factory's token, deploy in seconds — no Kubernetes namespace creation, no per-tenant Helm chart. Each assistant is isolated at the URL and DB level, which is the only isolation B2B buyers actually verify.

## Related cases

- [AgentCore tenant-scoped spawning](/use-cases/agentcore-tenant-scoped-spawning) — lighter-weight tenant runtime without the chatbot UI
- [Cloudflare sub-agent factory per user](/use-cases/cloudflare-sub-agent-factory-per-user) — per-user variant of the same factory pattern
- [Inbox-zero agent fleet](/use-cases/inbox-zero-agent-fleet) — per-user durable-agent shape adjacent to per-tenant
