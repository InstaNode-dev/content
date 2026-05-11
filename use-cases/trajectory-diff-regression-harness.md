---
title: Trajectory diff regression harness
category: P. Agent benchmarking & evaluation
services: ["minio", "pg"]
scenario: On every PR to the agent repo, a CI agent re-runs 1000 cached trajectories in parallel and stores diffs against the baseline in MinIO so reviewers see exactly which behaviors changed.
---
