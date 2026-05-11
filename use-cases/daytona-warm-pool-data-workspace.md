---
title: Daytona warm-pool data workspace
category: K. Ephemeral agent runtimes
services: ["redis", "deploy"]
scenario: A Daytona warm pool of 50 Python sandboxes each pre-attaches to a per-sandbox Redis namespace for caching pip-resolver results, so cold start to first agent action is under 100ms.
---
