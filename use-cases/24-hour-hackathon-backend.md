---
title: 24-hour hackathon backend
category: I. Hackathon & education
services: ["pg", "mongo", "minio"]
scenario: A team provisions Postgres + Mongo + MinIO anonymously, ships their demo, and lets the stack expire 24 hours after judging.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a team provisions Postgres + Mongo + MinIO anonymously, ships their demo, and lets the stack expire 24 hours after judging.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (Postgres + MongoDB + MinIO (S3-compatible)) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're scaffolding a 24-hour hackathon project. Provision a Postgres for users + submissions, a MongoDB for unstructured judge feedback, and a MinIO bucket for demo videos — all anonymously, no signup. Wire them into a Next.js app, deploy a demo, and stop worrying about cleanup since the stack auto-expires 24 hours after we provision.
```

## Steps to follow

- **Step 1: Claim three anonymous resources in parallel.** One curl per service, no auth, no account.

  ```bash
  curl -sX POST https://api.instanode.dev/db/new     > pg.json &
  curl -sX POST https://api.instanode.dev/nosql/new  > mongo.json &
  curl -sX POST https://api.instanode.dev/storage/new > minio.json &
  wait
  ```

- **Step 2: Export connection strings to the dev shell.** Each response has `connection_url` and `expires_at`.

  ```bash
  export DATABASE_URL=$(jq -r .connection_url pg.json)
  export MONGO_URL=$(jq -r .connection_url mongo.json)
  export S3_URL=$(jq -r .connection_url minio.json)
  ```

- **Step 3: Create the submissions schema.** Standard Postgres, real DB.

  ```sql
  CREATE TABLE submissions (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    team text NOT NULL,
    video_key text,
    submitted_at timestamptz DEFAULT now()
  );
  ```

- **Step 4: Upload demo videos to MinIO.** S3-compatible, works with any AWS SDK.

  ```bash
  aws --endpoint-url "$S3_URL" s3 cp demo.mp4 s3://instant-shared/team42/demo.mp4
  ```

- **Step 5: Ship and forget.** All three resources delete themselves at `expires_at` (T+24h). No teardown script, no abandoned infra in someone's AWS account.

## Why this works on instanode.dev

Hackathons die from cleanup debt — nobody remembers to tear down the RDS instance after Saturday night. The anonymous tier hands you real Postgres, Mongo, and S3 in one second each, then garbage-collects itself after 24 hours. Three curl calls replace three signup flows, three credit cards, and three Terraform configs.
