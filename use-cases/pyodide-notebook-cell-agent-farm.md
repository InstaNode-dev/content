---
title: Pyodide notebook cell agent farm
category: M. Parallel tool execution
services: ["minio", "nats"]
scenario: An analyst agent splits a 200-cell notebook across 30 Pyodide workers, each writing intermediate dataframes to MinIO so the planner can pull only the ones it needs.
---
