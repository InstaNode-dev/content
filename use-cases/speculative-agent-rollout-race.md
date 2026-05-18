---
title: Speculative agent rollout race
category: M. Parallel tool execution
services: ["nats", "pg"]
scenario: A speculative-decoding-style orchestrator runs the same task at three temperatures in parallel; a verifier agent picks the best output and discards the rest, all coordinated via NATS request/reply.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a speculative-decoding-style orchestrator runs the same task at three temperatures in parallel; a verifier agent picks the best output and discards the rest, all coordinated via NATS request/reply.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Run the same task three times in parallel at temperatures 0.2/0.7/1.0. Use NATS request/reply on subject "rollout.race.<task_id>" to dispatch and collect, then a verifier agent reads all three replies, picks the winner via a 3-way LLM judge, persists the choice + trajectories in Postgres. Provision both via instanode.dev.
```

## Steps to follow

- **Step 1: Provision NATS + Postgres.**

  ```bash
  NATS=$(curl -sX POST https://api.instanode.dev/queue/new -H 'Content-Type: application/json' -d '{"name":"speculative-agent-rollout-race-queue"}' | jq -r .connection_url)
  DB=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"speculative-agent-rollout-race-db"}' | jq -r .connection_url)
  ```

- **Step 2: Dispatch three rollouts in parallel.**

  ```python
  async with nats.connect(NATS) as nc:
      tasks = [nc.request(f"rollout.race.{task_id}",
                          json.dumps({"task": task, "temp": t}).encode(),
                          timeout=60) for t in (0.2, 0.7, 1.0)]
      replies = await asyncio.gather(*tasks)
  ```

- **Step 3: Verifier picks the winner.**

  ```python
  winner_idx = judge_model(task, [r.data.decode() for r in replies])
  ```

- **Step 4: Journal trajectories.**

  ```sql
  CREATE TABLE rollouts (
    task_id TEXT, temp REAL, trajectory JSONB,
    is_winner BOOL, judge_reasoning TEXT
  );
  INSERT INTO rollouts SELECT ...;
  ```

- **Step 5: Return the winning trajectory** as if it were the only one run.

## Why this works on instanode.dev

NATS request/reply gives synchronous-feeling fan-out across N workers — exactly the abstraction speculative decoding needs at the agent level. Recording trajectories in Postgres lets you mine the dataset later for "when does high-temperature actually win" without re-running.

## Related cases

- [Multi-model bake-off router](/use-cases/multi-model-bake-off-router) — races across providers instead of temperatures
- [Anthropic parallel tool_use batch](/use-cases/anthropic-parallel-tool-use-batch) — the same race-and-pick shape at the tool level
- [LLM-as-judge consensus pool](/use-cases/llm-as-judge-consensus-pool) — verifier variant that picks via consensus, not first-to-finish
