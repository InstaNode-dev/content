---
title: Trajectory diff regression harness
category: P. Agent benchmarking & evaluation
services: ["minio", "pg"]
scenario: On every PR to the agent repo, a CI agent re-runs 1000 cached trajectories in parallel and stores diffs against the baseline in MinIO so reviewers see exactly which behaviors changed.
---

## Sample agent prompt

```
On every PR to the agent repo, replay 1000 cached trajectories in parallel against the new code. For each, compare the new trajectory step-by-step to the baseline cached in MinIO, write a unified diff back to MinIO under prs/<pr>/<task>.diff, and log the per-PR aggregate (added/removed/changed steps) to Postgres. Provision both via instanode.dev.
```

## Steps to follow

- **Step 1: Provision MinIO + Postgres.**

  ```bash
  BUCKET=$(curl -sX POST https://api.instanode.dev/storage/new | jq -r .bucket)
  DB=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
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
