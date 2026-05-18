---
title: Cross-agent replay debugger
category: N. Multi-agent observability
services: ["storage", "mongo"]
scenario: A debug agent stores full prompt+response payloads for an entire multi-agent run in S3-compatible storage, with an index in Mongo so an engineer can replay any branch deterministically.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a debug agent stores full prompt+response payloads for an entire multi-agent run in S3-compatible storage, with an index in Mongo so an engineer can replay any branch deterministically.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (S3-compatible storage + MongoDB) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build me a replay debugger. Claim S3-compatible storage + Mongo on instanode.dev. On every LLM call in a multi-agent run, write the full request+response payload to S3-compatible storage keyed by run_id/turn/agent. Write an index doc to Mongo with the S3 key, parent turn, and outcome. Let me query "show me everything agent X said in run Y" and stream replays.
```

## Steps to follow

- **Step 1: Claim object storage and index.** S3-compatible S3-compatible storage for payloads, Mongo for the lookup graph.

  ```bash
  S3=$(curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"cross-agent-replay-debugger-storage"}')
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new -H 'Content-Type: application/json' -d '{"name":"cross-agent-replay-debugger-mongo"}' | jq -r .connection_url)
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

S3-compatible storage stores cheap, fat payload blobs; Mongo carries the cheap, fast lookup graph. The split avoids paying Postgres prices for blob storage and Mongo prices for binary data. Both come from the same anonymous token, so debug infra is one provisioning step — not an IAM ticket and a managed-Mongo trial.

## Related cases

- [Agent-run lineage store](/use-cases/agent-run-lineage-store) — the parent/child relation graph this debugger walks
- [OpenTelemetry agent-trace ingest](/use-cases/opentelemetry-agent-trace-ingest) — OTel collector that ingests the spans being replayed
- [Trajectory diff regression harness](/use-cases/trajectory-diff-regression-harness) — diffs cached replays of the same kind across versions
