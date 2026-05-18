---
title: Terminal-Bench shell sandbox grid
category: P. Agent benchmarking & evaluation
services: ["pg", "webhook", "deploy"]
scenario: A grid runner deploys 100 ephemeral shell sandboxes, each running a Terminal-Bench task with its own ephemeral DB; pass/fail webhooks aggregate to a Postgres results table.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a grid runner deploys 100 ephemeral shell sandboxes, each running a Terminal-Bench task with its own ephemeral DB; pass/fail webhooks aggregate to a Postgres results table.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + webhook receiver + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Spin up 100 ephemeral shell sandboxes, one per Terminal-Bench task, each with its own throwaway Postgres provisioned via instanode.dev. Each sandbox container runs the task; on completion it POSTs pass/fail to a shared webhook URL; a collector drains the webhook into a Postgres results table. Deploy + Postgres + webhook all via instanode.dev.
```

## Steps to follow

- **Step 1: One collector webhook + one shared results DB.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"terminal-bench-shell-sandbox-grid-webhook"}' | jq -r .receive_url)
  RESULTS=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"terminal-bench-shell-sandbox-grid-db"}' | jq -r .connection_url)
  ```

- **Step 2: For each task, fresh sandbox + scratch Postgres.**

  ```bash
  for task in $(ls tasks/); do
    SCRATCH=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"terminal-bench-shell-sandbox-grid-db"}' | jq -r .connection_url)
    curl -X POST https://api.instanode.dev/deploy/new \
      -H "Authorization: Bearer $INSTANODE_TOKEN" \
      -F "name=tbench-$task" \
      -F "image=tbench/runner:latest" \
      -F "env.TASK=$task" \
      -F "env.DB=$SCRATCH" \
      -F "env.REPORT=$WH"
  done
  ```

- **Step 3: Sandbox reports on exit.**

  ```bash
  curl -X POST "$REPORT" \
    -H "Content-Type: application/json" \
    -d "{\"task\":\"$TASK\",\"passed\":$EXIT_CODE,\"log\":\"$(tail -c 4096 /tmp/run.log | base64)\"}"
  ```

- **Step 4: Collector drains the webhook.**

  ```sql
  CREATE TABLE results (task TEXT PRIMARY KEY, passed BOOL, log TEXT, reported_at TIMESTAMPTZ);
  ```

- **Step 5: Grid pass-rate query** when the run is done.

## Why this works on instanode.dev

Terminal-Bench needs strict per-task isolation — one shell escaping into another invalidates the benchmark. `/deploy/new` per task plus `/db/new` per task gives that without managing 100 namespaces yourself. The webhook acts as a serverless results aggregator.

## Related cases

- [SWE-bench parallel rollout harness](/use-cases/swe-bench-parallel-rollout-harness) — code-task sibling at the same fleet scale
- [Sandboxed test runner per task](/use-cases/sandboxed-test-runner-per-task) — single-sandbox primitive this grid replicates 100x
- [E2B microVM sandbox per agent turn](/use-cases/e2b-microvm-sandbox-per-agent-turn) — lighter-weight per-turn sandboxing alternative
