---
title: Parallel SQL-plan probe
category: M. Parallel tool execution
services: ["pg", "redis"]
scenario: A data-analyst agent issues 12 candidate EXPLAIN ANALYZE queries against a forked Postgres in parallel, picking the lowest-cost plan to run against the real DB.
---
