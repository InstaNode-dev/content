---
title: Smithery one-MCP-per-skill mint
category: L. Agent-factory / spawning patterns
services: ["webhook", "redis", "deploy"]
scenario: A Smithery-like MCP host launches a dedicated MCP server per installed skill behind a webhook URL, each receiving tool calls and writing results to a shared Redis bus.
---

## Sample agent prompt

```
For each skill in the marketplace registry, mint a dedicated MCP server: deploy the skill's container via instanode.dev /deploy/new, register its webhook URL for tool invocations, and have it publish results to a shared Redis stream "mcp.results" the parent agent reads. One skill = one container = one webhook URL.
```

## Steps to follow

- **Step 1: Provision the shared Redis bus.**

  ```bash
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new | jq -r .connection_url)
  ```

- **Step 2: For each skill, deploy + webhook.**

  ```bash
  for skill in $(ls skills/); do
    DEPLOY=$(curl -sX POST https://api.instanode.dev/deploy/new \
      -d "{\"image\":\"ghcr.io/marketplace/$skill:latest\",\"env\":{\"REDIS_URL\":\"$REDIS\"}}")
    WH=$(curl -sX POST https://api.instanode.dev/webhook/new)
    register_skill "$skill" "$(echo $DEPLOY | jq -r .url)" "$(echo $WH | jq -r .receive_url)"
  done
  ```

- **Step 3: Skill receives a tool call, writes to Redis.**

  ```python
  result = run_skill_handler(payload)
  r.xadd("mcp.results", {"skill": SKILL_NAME, "call_id": call_id, "result": json.dumps(result)})
  ```

- **Step 4: Parent agent consumes the stream.**

  ```python
  for _, msgs in r.xread({"mcp.results": "$"}, block=5000):
      for msg_id, fields in msgs: dispatch(fields)
  ```

- **Step 5: Unminted skills cost nothing** — only deployed skills consume the tier quota.

## Why this works on instanode.dev

Each skill is fully isolated in its own container, but the shared Redis stream means the parent agent reads a single consumer-grouped firehose instead of polling N webhook URLs. `/deploy/new` + `/webhook/new` + `/cache/new` together form the entire Smithery substrate in three curl commands.
