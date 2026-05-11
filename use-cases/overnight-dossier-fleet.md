---
title: Overnight dossier fleet
category: Q. Background/async agent fleets
services: ["mongo", "minio"]
scenario: An async research service queues hundreds of overnight dossier jobs; workers persist intermediate findings in Mongo and PDF outputs in MinIO when complete.
---

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
