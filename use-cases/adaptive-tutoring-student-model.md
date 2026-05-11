---
title: Adaptive-tutoring student model
category: C. Vertical AI apps
services: ["pg"]
scenario: A tutor agent tracks per-student concept mastery, picks the next problem, and adjusts difficulty based on response patterns.
---

## Sample agent prompt

```
You're an adaptive math tutor. For each student, track concept-level mastery in Postgres (concept_id, attempts, correct, last_seen, EWMA score). Before serving a problem, query the model to pick the concept with the lowest mastery that's also due for review. Update mastery on every response.
```

## Steps to follow

- **Step 1: Provision the student-model database.**

  ```bash
  curl -sX POST https://api.instanode.dev/db/new \
    -H "Authorization: Bearer $INSTANT_TOKEN" \
    | jq -r .connection_url > .env.db
  ```

- **Step 2: Schema with one row per (student, concept).**

  ```sql
  CREATE TABLE mastery (
    student_id uuid,
    concept_id text,
    attempts int DEFAULT 0,
    correct int DEFAULT 0,
    ewma_score real DEFAULT 0.5,
    last_seen timestamptz,
    PRIMARY KEY (student_id, concept_id)
  );
  CREATE INDEX idx_due ON mastery (student_id, ewma_score, last_seen);
  ```

- **Step 3: Pick the next problem — weakest concept, spaced-repetition due.**

  ```sql
  SELECT concept_id
  FROM mastery
  WHERE student_id = $1
    AND (last_seen IS NULL OR last_seen < now() - interval '6 hours')
  ORDER BY ewma_score ASC, last_seen ASC NULLS FIRST
  LIMIT 1;
  ```

- **Step 4: Update on response with EWMA (alpha=0.3).**

  ```sql
  INSERT INTO mastery (student_id, concept_id, attempts, correct, ewma_score, last_seen)
  VALUES ($1, $2, 1, $3::int, $3::real, now())
  ON CONFLICT (student_id, concept_id) DO UPDATE
  SET attempts = mastery.attempts + 1,
      correct  = mastery.correct + EXCLUDED.correct,
      ewma_score = 0.7 * mastery.ewma_score + 0.3 * EXCLUDED.correct,
      last_seen = now();
  ```

## Why this works on instanode.dev

Adaptive tutoring needs real SQL for the ranking query (ORDER BY two columns with NULLS FIRST) and ACID guarantees so a double-tap doesn't double-update EWMA. Firebase can't express the "weakest, due, not seen in 6h" query; DynamoDB needs three GSIs. One curl, one schema, real Postgres — and the same DB scales from 5 pilot students to 50k without re-architecture.
