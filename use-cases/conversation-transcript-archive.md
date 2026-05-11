---
title: Conversation transcript archive
category: A. AI coding agents
services: ["minio"]
scenario: Coding sessions are persisted as JSONL transcripts for replay, fine-tuning, and "what did we try last sprint" search across the team.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: coding sessions are persisted as JSONL transcripts for replay, fine-tuning, and "what did we try last sprint" search across the team.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MinIO (S3-compatible)) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Persist every coding session as a JSONL transcript to MinIO on instanode.dev. One object per session, keyed by date + agent ID. On a "what did we try last sprint" query, list objects in the last 14 days, stream them, and grep for the topic.
```

## Steps to follow

- **Step 1: Claim S3-compatible storage.** Returns a bucket URL + IAM access key scoped to this token.

  ```bash
  curl -sX POST https://api.instanode.dev/storage/new | tee s3.json
  export AWS_ACCESS_KEY_ID=$(jq -r .access_key s3.json)
  export AWS_SECRET_ACCESS_KEY=$(jq -r .secret_key s3.json)
  export AWS_ENDPOINT_URL=$(jq -r .endpoint s3.json)
  export BUCKET=$(jq -r .bucket s3.json)
  ```

- **Step 2: At session end, upload the JSONL.** One object per session, partitioned by date.

  ```bash
  KEY="transcripts/$(date +%Y-%m-%d)/agent-${AGENT_ID}-$(date +%s).jsonl"
  aws s3 cp session.jsonl s3://$BUCKET/$KEY --endpoint-url $AWS_ENDPOINT_URL
  ```

- **Step 3: List recent sessions.** Standard S3 list — works with boto3, aws-cli, anything.

  ```python
  s3 = boto3.client("s3", endpoint_url=os.environ["AWS_ENDPOINT_URL"])
  resp = s3.list_objects_v2(Bucket=bucket, Prefix=f"transcripts/{since}/")
  ```

- **Step 4: Grep across the archive.** Stream + filter without downloading.

  ```bash
  aws s3 cp s3://$BUCKET/$KEY - --endpoint-url $AWS_ENDPOINT_URL | jq 'select(.content | contains("auth refactor"))'
  ```

## Why this works on instanode.dev

MinIO speaks the S3 API verbatim — every existing tool (aws-cli, boto3, rclone, duckdb httpfs) works unchanged. You get a real bucket with a real IAM user scoped to your token, not a stub. No AWS account, no IAM policy authoring, no bucket-name uniqueness collision; the bucket is provisioned and ready in ~600ms.

## Related cases

- [AutoGen group-chat history](/use-cases/autogen-group-chat-history.md) — the Mongo-backed online-replay alternative to JSONL archives
- [Cross-device chat history](/use-cases/cross-device-chat-history.md) — live-state version of the same transcript record
- [Cross-agent replay debugger](/use-cases/cross-agent-replay-debugger.md) — indexes transcripts like these for deterministic branch replay
