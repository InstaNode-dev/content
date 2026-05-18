---
title: Federated-learning aggregator
category: R. Edge agents & federated swarms
services: ["webhook", "storage", "pg"]
scenario: Edge agents on user devices compute private gradients and POST them to an aggregator webhook; the aggregator stores rounds in S3-compatible storage and the new global weights in Postgres.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: edge agents on user devices compute private gradients and POST them to an aggregator webhook; the aggregator stores rounds in S3-compatible storage and the new global weights in Postgres.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + S3-compatible storage + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a federated-learning aggregator. Claim a webhook receiver, a S3-compatible bucket, and a Postgres on instanode.dev. Edge agents POST gradient blobs to the webhook with a round_id header. The aggregator pulls blobs, FedAvgs them, writes new global weights into Postgres, and uploads the round artifact to S3-compatible storage for audit.
```

## Steps to follow

- **Step 1: Provision all three.** One per service.

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"federated-learning-aggregator-webhook"}' | jq -r .receive_url)
  S3=$(curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"federated-learning-aggregator-storage"}' | tee s3.json)
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"federated-learning-aggregator-db"}' | jq -r .connection_url)
  ```

- **Step 2: Edge agent uploads gradient.** Lightweight POST, no auth handshake.

  ```python
  httpx.post(WEBHOOK_URL,
             content=pickle.dumps(local_gradient),
             headers={"x-round": "42", "x-device": device_id, "Content-Type": "application/octet-stream"})
  ```

- **Step 3: Aggregator FedAvgs.** Pull all blobs for the round, average, write back.

  ```python
  reqs = httpx.get(f"https://api.instanode.dev/api/v1/webhooks/{token}/requests?since={round_start}").json()
  grads = [pickle.loads(base64.b64decode(r["body_raw"])) for r in reqs if r["headers"]["x-round"]=="42"]
  global_w = fedavg(grads)
  cur.execute("INSERT INTO global_weights (round, weights_bytea) VALUES (42, %s)", (pickle.dumps(global_w),))
  ```

- **Step 4: Archive the round for audit.** Reproducibility + compliance.

  ```bash
  aws s3 cp round-42-artifacts.tar.gz s3://$BUCKET/rounds/42/ --endpoint-url $S3_ENDPOINT
  ```

## Why this works on instanode.dev

Edge devices can't open inbound ports — they have to POST out. instanode's webhook URLs are real internet endpoints, accepting from anywhere, so phones in 50 countries can contribute gradients without VPN. Postgres holds the canonical small global state; S3-compatible storage swallows the large per-round artifacts. Three services, one token, no AWS bill to predict.

## Related cases

- [Per-device edge-agent state sync](/use-cases/per-device-edge-agent-state-sync) — edge-fleet sibling that syncs SQLite deltas instead of gradients
- [Geo-sharded chat-agent fleet](/use-cases/geo-sharded-chat-agent-fleet) — another partial-state-leaves-the-edge replication pattern
- [OpenTelemetry agent-trace ingest](/use-cases/opentelemetry-agent-trace-ingest) — similar webhook+Postgres+S3-compatible storage ingestion at agent scale
