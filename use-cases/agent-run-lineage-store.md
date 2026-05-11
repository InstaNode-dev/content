---
title: Agent-run lineage store
category: N. Multi-agent observability
services: ["pg", "minio"]
scenario: An evaluation team builds a lineage UI where every run links to its parent run and its eval verdict; the relation graph lives in Postgres and the raw transcripts in MinIO.
---

## Sample agent prompt

```
You're building an evaluation lineage UI. Every agent run has a parent_run_id, an eval verdict (pass/fail/partial), and a raw transcript JSON that's too big to embed inline. Store the lineage graph in Postgres and the transcripts in MinIO. Surface "show me the failure chain for run X" as a single recursive query.
```

## Steps to follow

- **Step 1: Provision both stores.**

  ```bash
  PG=$(curl -sX POST https://api.instanode.dev/db/new -H "Authorization: Bearer $INSTANT_TOKEN" | jq -r .connection_url)
  S3=$(curl -sX POST https://api.instanode.dev/storage/new -H "Authorization: Bearer $INSTANT_TOKEN")
  ```

- **Step 2: Lineage table with self-FK.**

  ```sql
  CREATE TABLE runs (
    id uuid PRIMARY KEY,
    parent_id uuid REFERENCES runs(id),
    agent_name text NOT NULL,
    verdict text CHECK (verdict IN ('pass','fail','partial','pending')),
    transcript_key text NOT NULL,
    created_at timestamptz DEFAULT now()
  );
  CREATE INDEX idx_runs_parent ON runs (parent_id);
  ```

- **Step 3: Write a run — transcript to MinIO, metadata to Postgres.**

  ```python
  key = f"transcripts/{run_id}.json"
  s3.put_object(Bucket="instant-shared", Key=key, Body=json.dumps(transcript))
  pg.execute("INSERT INTO runs (id,parent_id,agent_name,verdict,transcript_key) VALUES (%s,%s,%s,%s,%s)",
             (run_id, parent_id, name, verdict, key))
  ```

- **Step 4: Recursive query — full ancestry of a failure.**

  ```sql
  WITH RECURSIVE ancestry AS (
    SELECT * FROM runs WHERE id = $1
    UNION ALL
    SELECT r.* FROM runs r JOIN ancestry a ON r.id = a.parent_id
  )
  SELECT id, agent_name, verdict, transcript_key FROM ancestry;
  ```

## Why this works on instanode.dev

Lineage is fundamentally relational — recursive CTEs are the right primitive, and embedding 100KB transcripts inline destroys query speed. Two curls give you the right tool for each half: Postgres for the graph, MinIO for the heavy JSON. Fetch the chain in one query, then presigned-URL the transcripts only when the UI expands a node.
