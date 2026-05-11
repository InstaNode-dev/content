---
title: Magentic-One DAG executor
category: J. Agent swarms & fan-out
services: ["nats", "pg"]
scenario: A Microsoft-Magentic-One-style orchestrator decomposes a goal into a task DAG, dispatches independent branches to parallel agents, and merges their outputs after a quality-gate agent approves each branch.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Microsoft-Magentic-One-style orchestrator decomposes a goal into a task DAG, dispatches independent branches to parallel agents, and merges their outputs after a quality-gate agent approves each branch.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Implement a Magentic-One-style orchestrator: decompose a goal into a DAG of tasks, dispatch independent branches to worker agents via NATS, and merge their outputs after a quality-gate agent approves each branch. Provision NATS for task dispatch and Postgres to hold the DAG and branch outputs.
```

## Steps to follow

- **Step 1: Provision queue and DB.**

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new -d '{"stream":"dag"}' -H 'Content-Type: application/json'
  curl -s -X POST https://api.instanode.dev/db/new
  ```

- **Step 2: DAG and outputs schema.**

  ```sql
  CREATE TABLE tasks (
    run_id uuid, task_id text, depends_on text[], status text DEFAULT 'pending',
    output jsonb, gate_passed boolean,
    PRIMARY KEY (run_id, task_id)
  );
  ```

- **Step 3: Orchestrator dispatches ready tasks.**

  ```python
  ready = pg.fetch("""
    SELECT task_id FROM tasks WHERE run_id=$1 AND status='pending'
    AND NOT EXISTS (SELECT 1 FROM unnest(depends_on) d
                    JOIN tasks t ON t.task_id=d WHERE t.status<>'gated_ok')""", run)
  for t in ready:
      await js.publish(f"dag.work.{t['task_id']}", json.dumps({...}).encode())
  ```

- **Step 4: Workers complete and push to a quality gate.**

  ```python
  async def worker(msg):
      out = run_agent(msg)
      pg.execute("UPDATE tasks SET output=$1, status='gating' WHERE task_id=$2", out, msg["task"])
      await js.publish("dag.gate", json.dumps({"task": msg["task"]}).encode())
  ```

- **Step 5: Gate agent approves or rejects.**

  ```python
  verdict = gate_llm.judge(output)
  pg.execute("UPDATE tasks SET gate_passed=$1, status=$2 WHERE task_id=$3",
             verdict.ok, "gated_ok" if verdict.ok else "rejected", task)
  ```

## Why this works on instanode.dev

The DAG state lives in Postgres so the orchestrator restarts cleanly, and NATS subjects let independent branches run truly in parallel without a workflow engine. One claimed token, two curls, and the gating loop is queryable SQL.
