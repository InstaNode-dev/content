---
title: Multi-model bake-off router
category: M. Parallel tool execution
services: ["redis", "nats"]
scenario: A router agent dispatches the same prompt to GPT, Claude, and Gemini concurrently; the first valid JSON response wins, and loser turns get cached in Redis for next time.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a router agent dispatches the same prompt to GPT, Claude, and Gemini concurrently; the first valid JSON response wins, and loser turns get cached in Redis for next time.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a model bake-off router. The router publishes the same prompt to GPT, Claude, and Gemini concurrently on NATS subjects, the first valid JSON reply wins, and the slower replies get cached in Redis so we never re-run them for the same prompt hash. Provision NATS and Redis.
```

## Steps to follow

- **Step 1: Provision the bus and cache.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"name":"multi-model-bake-off-router-queue","stream":"bakeoff"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"multi-model-bake-off-router-cache"}'
  ```

- **Step 2: Three model workers, queue-grouped by model name.**

  ```python
  await nc.subscribe("bakeoff.gpt", queue="gpt", cb=run_gpt)
  await nc.subscribe("bakeoff.claude", queue="claude", cb=run_claude)
  await nc.subscribe("bakeoff.gemini", queue="gemini", cb=run_gemini)
  ```

- **Step 3: Router fans out and races.**

  ```python
  inflight = [
      asyncio.create_task(nc.request(f"bakeoff.{m}", payload, timeout=15))
      for m in ("gpt", "claude", "gemini")
  ]
  done, pending = await asyncio.wait(inflight, return_when=asyncio.FIRST_COMPLETED)
  winner = next(iter(done)).result()
  ```

- **Step 4: Loser replies finish in the background, cached by prompt hash.**

  ```python
  async def stash(task, model):
      try:
          r = await task
          await redis.setex(f"cache:{model}:{prompt_hash}", 86400, r.data)
      except: pass
  for t, m in zip(pending, ("claude", "gemini")):
      asyncio.create_task(stash(t, m))
  ```

- **Step 5: Next time, peek the cache before dispatching.**

  ```python
  hit = await redis.get(f"cache:claude:{prompt_hash}")
  if hit: return json.loads(hit)
  ```

## Why this works on instanode.dev

NATS' first-reply-wins pattern falls out of the request/wait primitive without extra plumbing, and Redis keeps the "wasted" generations productive by feeding the next request. Both resources are one curl each.

## Related cases

- [Anthropic parallel tool_use batch](/use-cases/anthropic-parallel-tool-use-batch.md) — same race-and-collect shape, but across tools instead of models
- [Speculative agent rollout race](/use-cases/speculative-agent-rollout-race.md) — races temperatures of one model instead of three providers
- [LLM-as-judge consensus pool](/use-cases/llm-as-judge-consensus-pool.md) — judge variant when you want consensus instead of first-to-win
