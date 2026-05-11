---
title: Federated-learning aggregator
category: R. Edge agents & federated swarms
services: ["webhook", "minio", "pg"]
scenario: Edge agents on user devices compute private gradients and POST them to an aggregator webhook; the aggregator stores rounds in MinIO and the new global weights in Postgres.
---

## Sample agent prompt

```
Build a federated-learning aggregator. Claim a webhook receiver, a MinIO bucket, and a Postgres on instanode.dev. Edge agents POST gradient blobs to the webhook with a round_id header. The aggregator pulls blobs, FedAvgs them, writes new global weights into Postgres, and uploads the round artifact to MinIO for audit.
```

## Steps to follow

- **Step 1: Provision all three.** One per service.

  ```bash
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  S3=$(curl -sX POST https://api.instanode.dev/storage/new | tee s3.json)
  PG=$(curl -sX POST https://api.instanode.dev/db/new | jq -r .connection_url)
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

Edge devices can't open inbound ports — they have to POST out. instanode's webhook URLs are real internet endpoints, accepting from anywhere, so phones in 50 countries can contribute gradients without VPN. Postgres holds the canonical small global state; MinIO swallows the large per-round artifacts. Three services, one token, no AWS bill to predict.
