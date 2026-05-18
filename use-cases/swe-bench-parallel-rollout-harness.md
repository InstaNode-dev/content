---
title: SWE-bench parallel rollout harness
category: P. Agent benchmarking & evaluation
services: ["pg", "storage", "deploy"]
scenario: A benchmark runner spawns 500 isolated agent instances against 500 SWE-bench-Verified tasks, each with a private Postgres scratch and its results written to S3-compatible storage for the judge model.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a benchmark runner spawns 500 isolated agent instances against 500 SWE-bench-Verified tasks, each with a private Postgres scratch and its results written to S3-compatible storage for the judge model.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + S3-compatible storage + container deploy) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Run SWE-bench-Verified end-to-end on 500 tasks in parallel. For each task: provision a Postgres scratch + deploy a per-task agent container via instanode.dev, mount the buggy repo, run the agent's patch loop, then write the final patch + test output to S3-compatible storage under runs/<run_id>/<task_id>/. Aggregate pass@1 into a results table.
```

## Steps to follow

- **Step 1: Provision shared S3-compatible storage + per-task Postgres in batch.**

  ```bash
  BUCKET=$(curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"swe-bench-parallel-rollout-harness-storage"}' | jq -r .bucket)
  for task in $(jq -r .[].instance_id verified.json); do
    curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"swe-bench-parallel-rollout-harness-db"}' \
      | jq --arg t "$task" '. + {task: $t}' >> dbs.jsonl
  done
  ```

- **Step 2: Deploy 500 isolated agent containers.**

  ```bash
  parallel -j 50 --colsep ' ' \
    'curl -X POST https://api.instanode.dev/deploy/new -H "Authorization: Bearer $INSTANODE_TOKEN" -F "name=swe-agent-{1}" -F "image=ghcr.io/me/agent:swe" -F "env.TASK={1}" -F "env.DB={2}" -F "env.S3_BUCKET=$BUCKET"' \
    :::: tasks-and-dbs.tsv
  ```

- **Step 3: Agent writes patch + test log to S3-compatible storage.**

  ```python
  s3.put_object(Bucket=BUCKET, Key=f"runs/{run_id}/{task_id}/patch.diff", Body=patch)
  s3.put_object(Bucket=BUCKET, Key=f"runs/{run_id}/{task_id}/test.log", Body=log)
  ```

- **Step 4: Judge model scores each.**

  ```sql
  INSERT INTO results(run_id, task_id, passed, reasoning) VALUES (...);
  ```

- **Step 5: Compute pass@1.**

  ```sql
  SELECT avg(passed::int) FROM results WHERE run_id = $1;
  ```

## Why this works on instanode.dev

500 separate Postgres scratches + 500 isolated deploys would take hours to set up on a cloud account. Here it's three loops of curl. The pro tier gives the connection ceiling and per-resource isolation that prevents one task's flaky migration from poisoning another.

## Related cases

- [GAIA tournament bracket](/use-cases/gaia-tournament-bracket.md) — tournament-scoring variant of the same parallel eval fleet
- [Terminal-Bench shell sandbox grid](/use-cases/terminal-bench-shell-sandbox-grid.md) — shell-task sibling at fleet scale with webhook pass/fail
- [Trajectory diff regression harness](/use-cases/trajectory-diff-regression-harness.md) — per-PR regression cousin of running the full benchmark
