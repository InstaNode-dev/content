---
title: Deploy-status MCP server
category: F. Developer tooling
services: ["nats", "redis"]
scenario: An MCP server tracks live deploy status across environments and pushes updates to a chat agent over pub/sub.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an MCP server tracks live deploy status across environments and pushes updates to a chat agent over pub/sub.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build an MCP server that exposes deploy_status across staging/prod. Use NATS for pushing live updates and Redis as the current-state cache. When a deploy starts/succeeds/fails, the CI runner publishes to NATS subject deploys.* and the MCP server updates a Redis hash; the chat agent reads via getDeployStatus tool.
```

## Steps to follow

- **Step 1: Provision NATS + Redis.** Two curls, anonymous tier.

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: CI runner publishes status events.** One line at each deploy lifecycle step.

  ```bash
  nats pub deploys.prod.api "{\"status\":\"deploying\",\"sha\":\"$GIT_SHA\"}" --server $NATS_URL
  ```

- **Step 3: MCP server subscribes + updates Redis.** Pub→cache→tool exposes status with <1s lag.

  ```python
  async def on_event(msg):
      env, svc = msg.subject.split(".")[1:3]
      r.hset(f"deploy:{env}:{svc}", mapping=json.loads(msg.data))
  await nc.subscribe("deploys.>", cb=on_event)
  ```

- **Step 4: Expose as MCP tool.** Chat agent reads current state in 1 call.

  ```python
  @mcp.tool
  def get_deploy_status(env: str, service: str) -> dict:
      return r.hgetall(f"deploy:{env}:{service}")
  ```

## Why this works on instanode.dev

NATS gives you push semantics — the chat agent finds out about a failed deploy without polling — while Redis is the canonical "current state" that survives restarts. Both provisioned anonymously means devs can spin this up before approving a paid plan, then claim and convert when the team wants it permanent.
