---
title: OpenTelemetry agent-trace ingest
category: N. Multi-agent observability
services: ["webhook", "pg", "minio"]
scenario: A Langfuse-style collector receives OTel spans from 200 concurrent agents via a webhook endpoint, writes them to Postgres, and stores large prompt payloads in MinIO.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a Langfuse-style collector receives OTel spans from 200 concurrent agents via a webhook endpoint, writes them to Postgres, and stores large prompt payloads in MinIO.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Postgres + MinIO (S3-compatible)) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a Langfuse-style OTel ingestor for our 200-agent swarm. Spans arrive on a webhook endpoint, get unpacked into Postgres for query, and large prompt/response bodies overflow to MinIO with a pointer back. Provision webhook, Postgres, and MinIO.
```

## Steps to follow

- **Step 1: Provision all three.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new
  curl -s -X POST https://api.instanode.dev/db/new
  curl -s -X POST https://api.instanode.dev/storage/new
  ```

- **Step 2: Spans table.**

  ```sql
  CREATE TABLE spans (
    trace_id text, span_id text PRIMARY KEY, parent_id text,
    name text, started_at timestamptz, duration_ms int,
    attributes jsonb, body_key text
  );
  CREATE INDEX ON spans (trace_id);
  CREATE INDEX ON spans (started_at DESC);
  ```

- **Step 3: Point the OTel SDK at the receiver URL.**

  ```python
  from opentelemetry.sdk.trace.export import BatchSpanProcessor
  from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
  exporter = OTLPSpanExporter(endpoint=f"{WH_URL}/v1/traces")
  ```

- **Step 4: Ingest worker offloads large bodies to MinIO.**

  ```python
  for span in fetch_webhook_batch():
      body = span["attributes"].pop("llm.prompt", None)
      key = None
      if body and len(body) > 8192:
          key = f"prompts/{span['trace_id']}/{span['span_id']}.txt"
          s3.put_object(Bucket="otel", Key=key, Body=body)
      pg.execute("INSERT INTO spans VALUES (...)", ..., key)
  ```

- **Step 5: Query a slow trace.**

  ```sql
  SELECT name, duration_ms, attributes FROM spans
  WHERE trace_id = $1 ORDER BY started_at;
  ```

## Why this works on instanode.dev

The webhook absorbs bursts when 200 agents flush at once; MinIO keeps Postgres rows skinny by holding the 50KB prompt bodies; the trace dashboard joins them on demand. One claimed token, three resources, zero observability vendor lock-in.
