---
title: LangGraph state checkpoints
category: B. Multi-agent systems
services: ["pg"]
scenario: A LangGraph workflow checkpoints node state to Postgres so a crashed run resumes mid-graph instead of restarting from scratch.
---

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
