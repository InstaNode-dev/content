---
title: Cross-agent replay debugger
category: N. Multi-agent observability
services: ["minio", "mongo"]
scenario: A debug agent stores full prompt+response payloads for an entire multi-agent run in MinIO, with an index in Mongo so an engineer can replay any branch deterministically.
---

## Sample agent prompt

```
Build me a replay debugger. Claim MinIO + Mongo on instanode.dev. On every LLM call in a multi-agent run, write the full request+response payload to MinIO keyed by run_id/turn/agent. Write an index doc to Mongo with the S3 key, parent turn, and outcome. Let me query "show me everything agent X said in run Y" and stream replays.
```

## Steps to follow

- **Step 1: Claim object storage and index.** S3-compatible MinIO for payloads, Mongo for the lookup graph.

  ```bash
  S3=$(curl -sX POST https://api.instanode.dev/storage/new)
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  ```

- **Step 2: Wrap every LLM call.** Persist the full payload and index it.

  ```python
  def trace(run_id, turn, agent, req, resp):
      key = f"{run_id}/{turn}/{agent}.json"
      s3.put_object(Bucket=bucket, Key=key, Body=json.dumps({"req":req,"resp":resp}))
      mongo.turns.insert_one({"run_id":run_id,"turn":turn,"agent":agent,"key":key,"parent":parent_turn})
  ```

- **Step 3: Query a run's branches.** Mongo's graph fields make tree traversal cheap.

  ```python
  pipeline = [{"$match": {"run_id": run_id}},
              {"$graphLookup": {"from":"turns","startWith":"$turn","connectFromField":"turn","connectToField":"parent","as":"descendants"}}]
  ```

- **Step 4: Replay deterministically.** Pull the payload, re-feed the LLM with temperature 0.

  ```python
  turn = mongo.turns.find_one({"run_id": run_id, "agent": "critic"})
  payload = json.loads(s3.get_object(Bucket=bucket, Key=turn["key"])["Body"].read())
  replay = openai.chat.completions.create(**payload["req"], temperature=0)
  ```

## Why this works on instanode.dev

MinIO stores cheap, fat payload blobs; Mongo carries the cheap, fast lookup graph. The split avoids paying Postgres prices for blob storage and Mongo prices for binary data. Both come from the same anonymous token, so debug infra is one provisioning step — not an IAM ticket and a managed-Mongo trial.
