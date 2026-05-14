---
title: High-volume PR-review pipeline
category: F. Developer tooling
services: ["nats", "minio"]
scenario: An automated reviewer handles thousands of MRs/day, queues each review job, and stores comment artifacts per run.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: an automated reviewer handles thousands of MRs/day, queues each review job, and stores comment artifacts per run.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (NATS JetStream + S3-compatible storage) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up the backend for our AI PR-review bot that handles 3k MRs/day. Provision a NATS JetStream queue for review jobs and a S3-compatible bucket for the markdown comment artifacts each run produces. Wire a small Go publisher that enqueues one job per merge_request.open webhook and a worker that reads from the stream, runs the review, and uploads the artifact. Return the connection URLs once both resources are live.
```

## Steps to follow

- **Step 1: Provision the JetStream queue.** One curl gets you a durable NATS URL with a stream pre-created.

  ```bash
  curl -s -X POST https://api.instanode.dev/queue/new \
    -H 'Content-Type: application/json' \
    -d '{"stream":"pr-reviews","subjects":["review.>"]}' | jq .
  ```

- **Step 2: Provision the artifact bucket.** A scoped S3-compatible bucket for per-MR markdown.

  ```bash
  curl -s -X POST https://api.instanode.dev/storage/new | jq -r .connection_url
  ```

- **Step 3: Publish a job on each GitLab webhook.**

  ```python
  import nats, json
  nc = await nats.connect(os.environ["NATS_URL"])
  js = nc.jetstream()
  await js.publish("review.queued", json.dumps({"mr_id": mr, "sha": sha}).encode())
  ```

- **Step 4: Pull-consumer in the worker.** Acks only on successful upload to S3-compatible storage.

  ```python
  sub = await js.pull_subscribe("review.queued", durable="reviewer")
  msgs = await sub.fetch(8, timeout=5)
  ```

- **Step 5: Upload the markdown comment file with the MR id as the key.**

  ```bash
  aws --endpoint-url "$MINIO_ENDPOINT" s3 cp review.md s3://artifacts/mr-$MR_ID.md
  ```

## Why this works on instanode.dev

JetStream gives you at-least-once delivery and durable consumers without standing up a broker, and S3-compatible storage is wire-compatible with the AWS CLI you already use. Both resources are claimed under the same token, so when traffic grows past the hobby tier you upgrade once and every consumer keeps its URL.

## Related cases

- [Devin-style PR-bot fleet](/use-cases/devin-style-pr-bot-fleet.md) — per-issue ephemeral-worker shape of the same review fleet
- [CI flake-tracker](/use-cases/ci-flake-tracker.md) — consumes the same CI webhook stream this pipeline reviews
- [PR-review bot triggered by webhooks](/use-cases/pr-review-bot-triggered-by-webhooks.md) — single-bot version of this thousands-of-PRs pipeline
