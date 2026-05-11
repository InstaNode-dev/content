---
title: OpenTelemetry agent-trace ingest
category: N. Multi-agent observability
services: ["webhook", "pg", "minio"]
scenario: A Langfuse-style collector receives OTel spans from 200 concurrent agents via a webhook endpoint, writes them to Postgres, and stores large prompt payloads in MinIO.
---
