---
title: Browser-agent action pool
category: M. Parallel tool execution
services: ["mongo", "nats"]
scenario: A Skyvern-style planner emits 20 actions (fill, click, scroll) that fan out across 20 Browserbase tabs simultaneously; results aggregate in Mongo keyed by step_id.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Skyvern-style planner emits 20 actions (fill, click, scroll) that fan out across 20 Browserbase tabs simultaneously; results aggregate in Mongo keyed by step_id.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're a Skyvern-style planner. After analyzing a target page, you emit 20 actions (fill, click, scroll, extract). Fan them out across 20 Browserbase tabs via NATS subjects, each tab posts its result to Mongo keyed by step_id. The planner aggregates results and moves to the next plan step. Don't wait sequentially.
```

## Steps to follow

- **Step 1: Provision the queue + result store.**

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new -H 'Content-Type: application/json' -d '{"name":"browser-agent-action-pool-queue"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new -H 'Content-Type: application/json' -d '{"name":"browser-agent-action-pool-mongo"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
  ```

- **Step 2: Planner publishes 20 actions, one subject per action type.**

  ```python
  for step in plan.steps:
      await nc.publish(f"browser.{step.action}",
          json.dumps({"step_id": step.id, "selector": step.selector,
                      "value": step.value, "tab_hint": step.tab}).encode())
  ```

- **Step 3: Browser worker — pull, act, persist.**

  ```python
  async def worker(sub_name):
      async for msg in nc.subscribe(f"browser.{sub_name}"):
          s = json.loads(msg.data)
          tab = await browserbase.session_for(s["tab_hint"])
          out = await execute(tab, s["selector"], s["value"])
          col.insert_one({"step_id": s["step_id"], "result": out,
                          "completed_at": datetime.utcnow()})
  ```

- **Step 4: Planner polls Mongo for completion of the batch.**

  ```python
  expected = set(s.id for s in plan.steps)
  while True:
      done = {d["step_id"] for d in col.find({"step_id": {"$in": list(expected)}})}
      if expected.issubset(done): break
      await asyncio.sleep(0.1)
  ```

## Why this works on instanode.dev

20 browser tabs in parallel is the hard part; the easy parts shouldn't slow you down. NATS subjects route actions to worker pools without an in-process scheduler; Mongo's keyed write is idempotent on step_id. Two curls and the data-plane disappears as a concern, leaving you to focus on the actual hard problem of recovering when a tab hits a CAPTCHA.

## Related cases

- [Accessibility-tree selector cache](/use-cases/accessibility-tree-selector-cache.md) — the selector-resolution layer underneath these parallel actions
- [Anthropic parallel tool_use batch](/use-cases/anthropic-parallel-tool-use-batch.md) — the same fan-out idea applied to model tool_use blocks
- [Browser job queue with retries](/use-cases/browser-job-queue-with-retries.md) — single-worker queue equivalent for navigation tasks
