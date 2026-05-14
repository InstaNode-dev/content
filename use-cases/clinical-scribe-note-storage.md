---
title: Clinical-scribe note storage
category: C. Vertical AI apps
services: ["pg", "minio"]
scenario: A medical scribe agent transcribes doctor-patient visits and stores structured SOAP notes per encounter with auditable history.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a medical scribe agent transcribes doctor-patient visits and stores structured SOAP notes per encounter with auditable history.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + S3-compatible storage) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're a clinical-scribe agent transcribing doctor-patient visits. For each visit, store structured SOAP fields (Subjective, Objective, Assessment, Plan) in Postgres for queryability, and the raw audio + full transcript blob in S3-compatible storage for audit. Every edit creates a new version row — never overwrite.
```

## Steps to follow

- **Step 1: Provision PG + bucket.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $T" | jq -r .connection_url)
  curl -sX POST https://api.instanode.dev/storage/new -H "Authorization: Bearer $T" > s3.json
  ```

- **Step 2: Versioned SOAP table.**

  ```sql
  CREATE TABLE soap_notes (
    note_id uuid NOT NULL,
    version int NOT NULL,
    encounter_id uuid NOT NULL,
    subjective text, objective text, assessment text, plan text,
    audio_key text, transcript_key text,
    authored_by text NOT NULL,
    authored_at timestamptz DEFAULT now(),
    PRIMARY KEY (note_id, version)
  );
  CREATE INDEX idx_encounter ON soap_notes (encounter_id, version DESC);
  ```

- **Step 3: Upload raw audio + transcript, insert metadata.**

  ```python
  audio_key = f"audio/{encounter_id}.opus"
  tx_key = f"transcripts/{encounter_id}.json"
  s3.put_object(Bucket="instant-shared", Key=audio_key, Body=audio_bytes)
  s3.put_object(Bucket="instant-shared", Key=tx_key, Body=json.dumps(transcript))
  pg.execute("""INSERT INTO soap_notes (note_id,version,encounter_id,
                subjective,objective,assessment,plan,audio_key,transcript_key,authored_by)
                VALUES (%s,1,%s,%s,%s,%s,%s,%s,%s,%s)""",
             (nid, encounter_id, S, O, A, P, audio_key, tx_key, "scribe-v3"))
  ```

- **Step 4: Latest version of a note.**

  ```sql
  SELECT * FROM soap_notes WHERE note_id = $1 ORDER BY version DESC LIMIT 1;
  ```

## Why this works on instanode.dev

Clinical data needs strict auditability — every edit visible, raw artifacts untouched. Versioned PG rows + immutable objects in S3-compatible storage give that without a custom CDC pipeline. Both resources encrypt at rest by default. Two curls; the same setup that's HIPAA-shaped on day one scales without a re-platform when you go from one clinic pilot to a real deployment.

## Related cases

- [Personal-injury demand letters](/use-cases/personal-injury-demand-letters.md) — another vertical-AI workflow with auditable versioned documents
- [Voice-memo capture pipeline](/use-cases/voice-memo-capture-pipeline.md) — the audio-capture front end that feeds into structured notes
- [EHR appointment webhook fan-in](/use-cases/ehr-appointment-webhook-fan-in.md) — the upstream event source that triggers a scribe session
