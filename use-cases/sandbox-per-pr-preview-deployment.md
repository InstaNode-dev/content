---
title: Sandbox-per-PR preview deployment
category: K. Ephemeral agent runtimes
services: ["pg", "minio", "deploy"]
scenario: A reviewer agent provisions a throwaway storage bucket plus Postgres tied to a PR number, seeds it from a snapshot, and the whole thing is torn down when the PR closes.
---
