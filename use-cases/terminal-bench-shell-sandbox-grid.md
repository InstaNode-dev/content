---
title: Terminal-Bench shell sandbox grid
category: P. Agent benchmarking & evaluation
services: ["pg", "webhook", "deploy"]
scenario: A grid runner deploys 100 ephemeral shell sandboxes, each running a Terminal-Bench task with its own ephemeral DB; pass/fail webhooks aggregate to a Postgres results table.
---

## Sample agent prompt

```
Spin up 100 ephemeral shell sandboxes, one per Terminal-Bench task, each with its own throwaway Postgres provisioned via instanode.dev. Each sandbox container runs the task; on completion it POSTs pass/fail to a shared webhook URL; a collector drains the webhook into a Postgres results table. Deploy + Postgres + webhook all via instanode.dev.
```

## Steps to follow

- **Step 1: One collector webhook + one shared results DB.**

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  RESULTS=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
  ```

- **Step 2: For each task, fresh sandbox + scratch Postgres.**

  ```bash
  for task in $(ls tasks/); do
    SCRATCH=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
    curl -X POST https://api.instanode.dev/deploy/new \
      -d "{\"image\":\"tbench/runner:latest\",\"env\":{\"TASK\":\"$task\",\"DB\":\"$SCRATCH\",\"REPORT\":\"$WH\"}}"
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
