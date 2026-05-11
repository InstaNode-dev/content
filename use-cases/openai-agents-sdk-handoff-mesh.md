---
title: OpenAI Agents SDK handoff mesh
category: J. Agent swarms & fan-out
services: ["pg", "redis"]
scenario: A triage agent hands off to specialist agents (refund, shipping, fraud) running concurrently, each maintaining its own conversation thread row keyed by handoff_id.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a triage agent hands off to specialist agents (refund, shipping, fraud) running concurrently, each maintaining its own conversation thread row keyed by handoff_id.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + Redis) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Use the OpenAI Agents SDK to wire a triage agent that hands off to refund, shipping, and fraud specialists. Each specialist runs concurrently and maintains its own thread keyed by handoff_id in Postgres. Use Redis for in-flight handoff locks to prevent duplicate work. Provision both and return URLs.
```

## Steps to follow

- **Step 1: Provision Postgres and Redis.**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new
  curl -s -X POST https://api.instanode.dev/cache/new
  ```

- **Step 2: Threads schema.**

  ```sql
  CREATE TABLE threads (
    handoff_id uuid PRIMARY KEY,
    specialist text NOT NULL,
    messages jsonb NOT NULL DEFAULT '[]',
    status text DEFAULT 'open',
    updated_at timestamptz DEFAULT now()
  );
  ```

- **Step 3: Lock + create thread on handoff.**

  ```python
  async def handoff(specialist, payload):
      hid = uuid4()
      if not await redis.set(f"lock:{hid}", "1", nx=True, ex=300):
          return None
      pg.execute("INSERT INTO threads(handoff_id, specialist) VALUES (%s,%s)", hid, specialist)
      return hid
  ```

- **Step 4: Wire the SDK with handoffs.**

  ```python
  from agents import Agent, handoff
  refund = Agent(name="refund", instructions="...")
  shipping = Agent(name="shipping", instructions="...")
  triage = Agent(name="triage", handoffs=[handoff(refund), handoff(shipping)])
  ```

- **Step 5: Append messages to the thread row.**

  ```sql
  UPDATE threads SET messages = messages || $1::jsonb, updated_at = now()
  WHERE handoff_id = $2;
  ```

## Why this works on instanode.dev

Postgres' `jsonb || jsonb` makes per-thread message appending atomic, and Redis NX locks stop the same handoff from being claimed twice when the triage agent retries. Two curls, both in the same token, no per-environment config drift.

## Related cases

- [CrewAI parallel-process crew](/use-cases/crewai-parallel-process-crew.md) — framework-equivalent specialist-agent fan-out
- [Cross-framework A2A gateway](/use-cases/cross-framework-a2a-gateway.md) — translates OpenAI SDK handoffs into other frameworks
- [LangGraph fan-out research agents](/use-cases/langgraph-fan-out-research-agents.md) — graph-shaped alternative to a flat handoff mesh
