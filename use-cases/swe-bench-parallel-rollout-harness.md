---
title: SWE-bench parallel rollout harness
category: P. Agent benchmarking & evaluation
services: ["pg", "minio", "deploy"]
scenario: A benchmark runner spawns 500 isolated agent instances against 500 SWE-bench-Verified tasks, each with a private Postgres scratch and its results written to MinIO for the judge model.
---
