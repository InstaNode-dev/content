---
title: Anthropic parallel tool_use batch
category: M. Parallel tool execution
services: ["nats", "redis"]
scenario: A Claude turn emits 8 simultaneous tool_use blocks; an executor fans them out to 8 NATS subjects, each handler writes its result to Redis keyed by tool_use_id, and the assistant collects them for the next turn.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Claude turn emits 8 simultaneous tool_use blocks; an executor fans them out to 8 NATS subjects, each handler writes its result to Redis keyed by tool_use_id, and the assistant collects them for the next turn.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
A Claude turn just returned 8 tool_use blocks in parallel. Fan them out to 8 NATS subjects, one per tool. Each handler writes its result to Redis keyed by tool_use_id with a 5-minute TTL. The orchestrator MGETs all 8 keys, assembles the tool_result blocks, and feeds them back into the next turn. Target: full fan-out + collection in under the slowest tool's latency.
```

## Steps to follow

- **Step 1: Provision NATS + Redis.**

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Fan out the 8 tool_use blocks.**

  ```python
  for block in response.content:
      if block.type == "tool_use":
          await nc.publish(f"tool.{block.name}",
                           json.dumps({"id": block.id, "input": block.input}).encode())
  ```

- **Step 3: Handlers stash results in Redis by tool_use_id.**

  ```python
  async def handle_search(msg):
      payload = json.loads(msg.data)
      result = await search_api(payload["input"]["query"])
      r.set(f"toolres:{payload['id']}", json.dumps(result), ex=300)
  ```

- **Step 4: Orchestrator collects with a single MGET, polling briefly.**

  ```python
  ids = [b.id for b in response.content if b.type == "tool_use"]
  deadline = time.time() + 30
  while time.time() < deadline:
      vals = r.mget([f"toolres:{i}" for i in ids])
      if all(vals):
          break
      await asyncio.sleep(0.05)
  tool_results = [{"type":"tool_result","tool_use_id":i,"content":json.loads(v)}
                  for i,v in zip(ids, vals)]
  ```

## Why this works on instanode.dev

Anthropic's parallel tool_use only pays off if your collection layer is faster than your tools. Redis MGET pulls 8 values in one round-trip (~1ms); NATS fan-out is sub-millisecond. Both provisioned in two curls, no Lambda cold-start, no SQS visibility-timeout gymnastics. Drop-in replacement when you outgrow in-process asyncio.gather.
