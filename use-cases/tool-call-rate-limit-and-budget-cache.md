---
title: Tool-call rate-limit and budget cache
category: A. AI coding agents
services: ["redis"]
scenario: A coding agent caches LLM token budgets and rate-limit windows in Redis so parallel sub-agents don't burn quota fighting each other.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a coding agent caches LLM token budgets and rate-limit windows in Redis so parallel sub-agents don't burn quota fighting each other.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Provision Redis via instanode.dev. Use it as a coordination layer so parallel sub-agents share a token budget and OpenAI tier-1 rate-limit windows. Every model call decrements a shared budget counter and acquires a rate-limit slot via a Redis Lua script; if either fails, the sub-agent waits.
```

## Steps to follow

- **Step 1: Provision Redis.**

  ```bash
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"tool-call-rate-limit-and-budget-ca-cache"}' | jq -r .connection_url)
  ```

- **Step 2: Atomic budget + rate-limit Lua script.**

  ```lua
  local budget = redis.call("DECRBY", KEYS[1], ARGV[1])
  if budget < 0 then redis.call("INCRBY", KEYS[1], ARGV[1]); return -1 end
  local rpm = redis.call("INCR", KEYS[2])
  if rpm == 1 then redis.call("EXPIRE", KEYS[2], 60) end
  if rpm > tonumber(ARGV[2]) then redis.call("INCRBY", KEYS[1], ARGV[1]); return -2 end
  return budget
  ```

- **Step 3: Sub-agent acquires before every call.**

  ```python
  acquire = r.register_script(LUA)
  result = acquire(keys=["budget:run-1", "rpm:openai:tier1"], args=[est_tokens, 500])
  if result < 0: time.sleep(1); continue
  ```

- **Step 4: On model response, settle.** If estimated > actual, refund the diff.

  ```python
  r.incrby(f"budget:run-1", est_tokens - actual_tokens)
  ```

- **Step 5: Single source of truth** — 30 sub-agents see one shared counter, no double-spend.

## Why this works on instanode.dev

Redis EVAL is the canonical "compare-and-decrement atomically" primitive for shared budgets — without it parallel sub-agents will overspend by exactly N-1 calls. `/cache/new` ships a Redis with ACL-scoped credentials so the agent's budget keys can't be touched by anything else.

## Related cases

- [Per-agent rate-limited API key vault](/use-cases/per-agent-rate-limited-api-key-vault.md) — vault that mints the keys this budget cache governs
- [Token-cost ledger per sub-agent](/use-cases/token-cost-ledger-per-sub-agent.md) — Postgres rollup that this Redis cache feeds
- [Contract redline cache](/use-cases/contract-redline-cache.md) — another Redis-fronted reuse cache that saves model spend
