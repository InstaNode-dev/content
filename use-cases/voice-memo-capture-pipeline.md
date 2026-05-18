---
title: Voice-memo capture pipeline
category: D. Personal AI
services: ["storage", "pg"]
scenario: A second-brain agent receives uploaded voice memos, transcribes them, and files transcripts plus audio for semantic search.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a second-brain agent receives uploaded voice memos, transcribes them, and files transcripts plus audio for semantic search.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (S3-compatible storage + Postgres) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Build a second-brain pipeline. Provision S3-compatible storage + Postgres via instanode.dev. When the user uploads a voice memo (via the app), store the original WAV in S3-compatible storage, run Whisper transcription, embed the transcript with text-embedding-3-small, and insert a row (audio_key, transcript, embedding, captured_at). Enable semantic search via pgvector.
```

## Steps to follow

- **Step 1: Provision both.**

  ```bash
  BUCKET=$(curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"voice-memo-capture-pipeline-storage"}' | jq -r .bucket)
  DB=$(curl -sX POST https://api.instanode.dev/db/new -H 'Content-Type: application/json' -d '{"name":"voice-memo-capture-pipeline-db"}' | jq -r .connection_url)
  ```

- **Step 2: Schema.**

  ```sql
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE memos (
    id BIGSERIAL PRIMARY KEY,
    audio_key TEXT, transcript TEXT,
    embedding vector(1536), captured_at TIMESTAMPTZ DEFAULT now()
  );
  CREATE INDEX ON memos USING hnsw (embedding vector_cosine_ops);
  ```

- **Step 3: Upload, transcribe, embed.**

  ```python
  key = f"memos/{uuid.uuid4()}.wav"
  s3.put_object(Bucket=BUCKET, Key=key, Body=audio_bytes)
  transcript = openai.audio.transcriptions.create(file=audio_bytes, model="whisper-1").text
  emb = openai.embeddings.create(input=transcript, model="text-embedding-3-small").data[0].embedding
  conn.execute("INSERT INTO memos(audio_key,transcript,embedding) VALUES (%s,%s,%s)",
               (key, transcript, emb))
  ```

- **Step 4: Semantic search.**

  ```sql
  SELECT transcript, audio_key FROM memos ORDER BY embedding <=> $1 LIMIT 10;
  ```

- **Step 5: Signed URL** for replay in the search results UI.

## Why this works on instanode.dev

The audio bytes belong in S3-compatible storage (cheap, S3-API), the transcript + embedding belong in Postgres (queryable, indexed). Two curls and the entire second-brain backend exists; pgvector handles cosine search natively without managing a vector DB on the side.

## Related cases

- [Clinical-scribe note storage](/use-cases/clinical-scribe-note-storage.md) — vertical-AI sibling that turns audio into structured records
- [Obsidian-vault embedding sync](/use-cases/obsidian-vault-embedding-sync.md) — downstream sink that can index transcripts for recall
- [arXiv-and-RSS research feed](/use-cases/arxiv-and-rss-research-feed.md) — another personal-AI ingestion pipeline keyed by webhook
