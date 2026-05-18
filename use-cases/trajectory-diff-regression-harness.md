---
title: Trajectory diff regression harness
category: P. Agent benchmarking & evaluation
services: ["storage", "pg"]
scenario: On every PR to the agent repo, a CI agent re-runs 1000 cached trajectories in parallel and stores diffs against the baseline in S3-compatible storage so reviewers see exactly which behaviors changed.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: on every PR to the agent repo, a CI agent re-runs 1000 cached trajectories in parallel and stores diffs against the baseline in S3-compatible storage so reviewers see exactly which behaviors changed.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (S3-compatible storage + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
On every PR to the agent repo, replay 1000 cached trajectories in parallel against the new code. For each, compare the new trajectory step-by-step to the baseline cached in S3-compatible storage, write a unified diff back to S3-compatible storage under prs/<pr>/<task>.diff, and log the per-PR aggregate (added/removed/changed steps) to Postgres. Provision both via instanode.dev.
```

## Steps to follow

- **Step 1: Provision S3-compatible storage + Postgres.**

  ```bash
  BUCKET=$(curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"trajectory-diff-regression-harness-storage"}' | jq -r .bucket)
  DB=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"trajectory-diff-regression-harness-db"}' | jq -r .connection_url)
  ```

- **Step 2: Replay against baseline.**

  ```python
  baseline = s3.get_object(Bucket=BUCKET, Key=f"baseline/{task}.json")["Body"].read()
  new = agent.run(task)
  diff = difflib.unified_diff(json.loads(baseline), new, lineterm="")
  ```

- **Step 3: Upload the diff.**

  ```python
  s3.put_object(Bucket=BUCKET, Key=f"prs/{pr_num}/{task}.diff", Body="\n".join(diff))
  ```

- **Step 4: Aggregate per PR.**

  ```sql
  CREATE TABLE pr_regressions(
    pr INT, task TEXT, steps_added INT, steps_removed INT, steps_changed INT
  );
  ```

- **Step 5: PR comment** links to a signed URL for each non-empty diff so the reviewer sees exactly which behaviors moved.

## Why this works on instanode.dev

A real S3-API bucket makes trajectory caching across CI runs trivial — no "did anyone clean the artifact cache" surprises. Postgres stores the structured per-PR rollup so trend dashboards can show "are diffs growing PR-over-PR" without re-parsing diff bodies.

## Related cases

- [SWE-bench parallel rollout harness](/use-cases/swe-bench-parallel-rollout-harness) — full-benchmark sibling of this per-PR regression check
- [Cross-agent replay debugger](/use-cases/cross-agent-replay-debugger) — manual-replay UI built on the same cached-trajectory pattern
- [Agent-run lineage store](/use-cases/agent-run-lineage-store) — lineage anchor used to attribute diffs to specific runs
