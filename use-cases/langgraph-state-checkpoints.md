---
title: LangGraph state checkpoints
category: B. Multi-agent systems
services: ["pg"]
scenario: A LangGraph workflow checkpoints node state to Postgres so a crashed run resumes mid-graph instead of restarting from scratch.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a LangGraph workflow checkpoints node state to Postgres so a crashed run resumes mid-graph instead of restarting from scratch.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Wire LangGraph's PostgresSaver checkpointer so our research workflow survives crashes. Provision Postgres on instanode, install the checkpointer schema, and configure the graph to save state after every node. Verify by killing the process mid-run and resuming from the same thread_id.
```

## Steps to follow

- **Step 1: Provision Postgres.**

  ```bash
  curl -s -X POST https://api.instanode.dev/db/new | jq -r .connection_url > .pg
  ```

- **Step 2: Install the checkpointer schema.**

  ```python
  from langgraph.checkpoint.postgres import PostgresSaver
  saver = PostgresSaver.from_conn_string(open(".pg").read().strip())
  saver.setup()
  ```

- **Step 3: Compile the graph with the checkpointer.**

  ```python
  graph = builder.compile(checkpointer=saver)
  cfg = {"configurable": {"thread_id": "research-2026-05-11-a"}}
  graph.invoke({"question": q}, cfg)
  ```

- **Step 4: Crash test.**

  ```bash
  # Kill the worker mid-node, then resume:
  python run.py --thread research-2026-05-11-a --resume
  ```

- **Step 5: Inspect the checkpoint trail.**

  ```sql
  SELECT thread_id, checkpoint_id, step, jsonb_path_query(checkpoint, '$.channel_values.messages')
  FROM checkpoints WHERE thread_id = 'research-2026-05-11-a' ORDER BY step;
  ```

## Why this works on instanode.dev

LangGraph's PostgresSaver only needs a working DSN, which the `/db/new` response gives you in one call. There is no separate checkpoint service to run, and the same DB row-stores your run history, so debugging a stuck thread is one SQL query away.

## Related cases

- [Form-fill state machine](/use-cases/form-fill-state-machine.md) — domain-specific instance of the same crash-resume pattern
- [Restate-style durable sidecar](/use-cases/restate-style-durable-sidecar.md) — out-of-process alternative to in-graph checkpoints
- [Long-horizon Temporal agent workflow](/use-cases/long-horizon-temporal-agent-workflow.md) — scales the checkpoint idea to 14-day workflows
