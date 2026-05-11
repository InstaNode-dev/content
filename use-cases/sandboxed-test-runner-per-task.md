---
title: Sandboxed test runner per task
category: A. AI coding agents
services: ["deploy"]
scenario: A coding agent spins up a throwaway container per task to run the user's test suite in isolation, tears it down on success or failure.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a coding agent spins up a throwaway container per task to run the user's test suite in isolation, tears it down on success or failure.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
For each task in tasks.jsonl, deploy a throwaway container via instanode.dev /deploy/new running the user's test suite. Mount their repo as a volume, run `pytest -x`, capture exit code and last 200 lines of stdout, then call /deploy/delete. Aggregate pass/fail to results.csv.
```

## Steps to follow

- **Step 1: Build the runner image once.**

  ```dockerfile
  FROM python:3.12-slim
  RUN pip install pytest pytest-json-report
  ENTRYPOINT ["pytest", "-x", "--json-report", "--json-report-file=/out/report.json"]
  ```

- **Step 2: Spawn one container per task.**

  ```bash
  for task in $(cat tasks.jsonl); do
    DEPLOY=$(curl -sX POST https://api.instanode.dev/deploy/new \
      -H "Content-Type: application/json" \
      -d "{\"image\":\"ghcr.io/me/runner:latest\",\"env\":{\"REPO\":\"$(echo $task | jq -r .repo)\"}}")
    echo "$DEPLOY" >> deploys.jsonl
  done
  ```

- **Step 3: Poll for completion.**

  ```bash
  while read d; do
    TOKEN=$(echo "$d" | jq -r .token)
    curl -s "https://api.instanode.dev/deploy/$TOKEN/logs" | tail -200
  done < deploys.jsonl
  ```

- **Step 4: Tear down each on exit.**

  ```bash
  curl -X DELETE "https://api.instanode.dev/deploy/$TOKEN"
  ```

- **Step 5: Aggregate** exit codes into results.csv for the planner agent to consume.

## Why this works on instanode.dev

`/deploy/new` returns in under a second and the deploy is fully isolated per token — no shared kernel state between tasks. Anonymous-tier deploys auto-expire in 24h, so a forgotten teardown won't leak a runner forever.

## Related cases

- [E2B microVM sandbox per agent turn](/use-cases/e2b-microvm-sandbox-per-agent-turn.md) — per-turn variant of the same throwaway-container idea
- [Daytona warm-pool data workspace](/use-cases/daytona-warm-pool-data-workspace.md) — warm-pool alternative when cold-start cost matters
- [Terminal-Bench shell sandbox grid](/use-cases/terminal-bench-shell-sandbox-grid.md) — fleet-scale version with 100 parallel sandboxes
