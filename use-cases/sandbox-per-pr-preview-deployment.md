---
title: Sandbox-per-PR preview deployment
category: K. Ephemeral agent runtimes
services: ["pg", "minio", "deploy"]
scenario: A reviewer agent provisions a throwaway storage bucket plus Postgres tied to a PR number, seeds it from a snapshot, and the whole thing is torn down when the PR closes.
---

## Sample agent prompt

```
On every PR opened against this repo, provision an isolated Postgres + MinIO bucket via instanode.dev, restore from the staging snapshot (s3://staging/snapshot.dump), and deploy the app container to pr-NNN.instanode.dev. On PR close, hit the delete endpoint for each resource. Pass the share URL back as a PR comment.
```

## Steps to follow

- **Step 1: Provision Postgres and MinIO for PR #42.**

  ```bash
  PR=42
  DB=$(curl -sX POST https://api.instanode.dev/db/new)
  BUCKET=$(curl -sX POST https://api.instanode.dev/storage/new)
  echo "$DB" > .preview/pr-$PR-db.json
  echo "$BUCKET" > .preview/pr-$PR-bucket.json
  ```

- **Step 2: Seed Postgres from snapshot.**

  ```bash
  pg_restore --clean --if-exists \
    -d "$(jq -r .connection_url .preview/pr-$PR-db.json)" \
    snapshots/staging.dump
  ```

- **Step 3: Mirror seed assets into the new bucket.**

  ```bash
  aws s3 sync s3://staging/fixtures/ \
    s3://$(jq -r .bucket .preview/pr-$PR-bucket.json)/ \
    --endpoint-url https://minio.instanode.dev
  ```

- **Step 4: Deploy the app.**

  ```bash
  curl -X POST https://api.instanode.dev/deploy/new \
    -d "{\"image\":\"ghcr.io/me/app:pr-$PR\",\"subdomain\":\"pr-$PR\"}"
  ```

- **Step 5: On PR close, fan-out deletes** using each resource's token.

## Why this works on instanode.dev

One token per resource means GitHub Actions can pass three opaque strings instead of three sets of credentials. `/storage/new` and `/db/new` complete in a single second each, so the preview is ready before CI finishes its first test.
