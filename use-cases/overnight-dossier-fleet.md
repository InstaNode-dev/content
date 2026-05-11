---
title: Overnight dossier fleet
category: Q. Background/async agent fleets
services: ["mongo", "minio"]
scenario: An async research service queues hundreds of overnight dossier jobs; workers persist intermediate findings in Mongo and PDF outputs in MinIO when complete.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an async research service queues hundreds of overnight dossier jobs; workers persist intermediate findings in Mongo and PDF outputs in MinIO when complete.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB + MinIO (S3-compatible)) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Spin up an overnight research fleet: queue 400 dossier jobs at 11pm, each worker persists intermediate findings to Mongo (flexible schema) and uploads the final PDF to MinIO. By 7am I want a manifest of completed dossiers with their PDF URLs. Provision Mongo and MinIO.
```

## Steps to follow

- **Step 1: Provision the stores.**

  ```bash
  curl -s -X POST https://api.instanode.dev/nosql/new
  curl -s -X POST https://api.instanode.dev/storage/new
  ```

- **Step 2: Seed the job collection.**

  ```python
  m["jobs"].insert_many([
      {"_id": str(uuid4()), "subject": s, "status": "queued"} for s in subjects
  ])
  m["jobs"].create_index([("status", 1)])
  ```

- **Step 3: Worker pulls a job, streams partials.**

  ```python
  job = m["jobs"].find_one_and_update({"status": "queued"}, {"$set": {"status": "running"}})
  for step in research(job["subject"]):
      m["partials"].insert_one({"job": job["_id"], "step": step.name, "data": step.data})
  ```

- **Step 4: Upload the PDF on completion.**

  ```python
  pdf_key = f"dossiers/{job['_id']}.pdf"
  s3.put_object(Bucket="reports", Key=pdf_key, Body=render_pdf(job))
  m["jobs"].update_one({"_id": job["_id"]}, {"$set": {"status": "done", "pdf": pdf_key}})
  ```

- **Step 5: Morning manifest.**

  ```javascript
  db.jobs.find({status: "done"}, {subject: 1, pdf: 1})
  ```

## Why this works on instanode.dev

Mongo's flexible docs hold the intermediate findings even when sub-agents return wildly different shapes, and MinIO holds the final PDFs cheaply. One token claims both; the morning manifest is a single `find()` away.
