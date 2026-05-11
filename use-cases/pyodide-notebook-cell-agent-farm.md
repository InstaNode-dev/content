---
title: Pyodide notebook cell agent farm
category: M. Parallel tool execution
services: ["minio", "nats"]
scenario: An analyst agent splits a 200-cell notebook across 30 Pyodide workers, each writing intermediate dataframes to MinIO so the planner can pull only the ones it needs.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an analyst agent splits a 200-cell notebook across 30 Pyodide workers, each writing intermediate dataframes to MinIO so the planner can pull only the ones it needs.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MinIO (S3-compatible) + NATS JetStream) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Split notebook.ipynb across 30 Pyodide workers. Each worker should pull its cell range from a NATS subject "cells.work", execute it in-browser, and write the resulting Arrow dataframe to a shared MinIO bucket keyed by cell_id. Provision the MinIO bucket and NATS via instanode.dev, then emit "cells.done" with the object key when each cell completes.
```

## Steps to follow

- **Step 1: Provision the bucket.** One curl for shared intermediate storage.

  ```bash
  curl -X POST https://api.instanode.dev/storage/new | tee storage.json
  export S3_URL=$(jq -r .connection_url storage.json)
  ```

- **Step 2: Provision NATS for work distribution.** JetStream gives durable, replayable cell assignments.

  ```bash
  curl -X POST https://api.instanode.dev/queue/new | tee nats.json
  export NATS_URL=$(jq -r .connection_url nats.json)
  ```

- **Step 3: Planner publishes 200 cell jobs.**

  ```python
  import nats, json
  nc = await nats.connect(NATS_URL)
  js = nc.jetstream()
  for i, cell in enumerate(notebook.cells):
      await js.publish("cells.work", json.dumps({"id": i, "src": cell.source}).encode())
  ```

- **Step 4: Each Pyodide worker pulls, executes, uploads.**

  ```python
  df.to_parquet(f"s3://{BUCKET}/cells/{cell_id}.parquet")
  await js.publish("cells.done", json.dumps({"id": cell_id, "key": f"cells/{cell_id}.parquet"}).encode())
  ```

- **Step 5: Planner pulls only the dataframes it actually needs** by listing `cells.done` and fetching the referenced keys.

## Why this works on instanode.dev

NATS JetStream is queue + durable log in one — workers can rejoin mid-run and replay missed cell assignments. MinIO is S3-API so the same `boto3` / `fsspec` code works in browser Pyodide and Node, and the bucket is provisioned in under a second with zero account setup.
