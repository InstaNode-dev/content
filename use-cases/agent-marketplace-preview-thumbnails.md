---
title: Agent-marketplace preview thumbnails
category: G. Internet-of-AI
services: ["minio"]
scenario: An agent marketplace stores screenshot previews of each listed agent's UI for human browsing.
---

## Sample agent prompt

```
You're indexing an agent marketplace. For every listed agent, capture a screenshot of its hosted UI and store it in MinIO under `thumbnails/{agent_id}.png`. Serve presigned URLs to the marketplace frontend so it can render a card grid without proxying images through your origin.
```

## Steps to follow

- **Step 1: Provision the bucket.**

  ```bash
  curl -sX POST https://api.instanode.dev/storage/new \
    -H "Authorization: Bearer $INSTANT_TOKEN" | jq .
  # returns connection_url, bucket, access_key, secret_key
  ```

- **Step 2: Capture and upload a thumbnail.**

  ```python
  import boto3
  from playwright.sync_api import sync_playwright

  s3 = boto3.client("s3", endpoint_url=ENDPOINT,
                    aws_access_key_id=AK, aws_secret_access_key=SK)

  with sync_playwright() as pw:
      page = pw.chromium.launch().new_page(viewport={"width":1280,"height":720})
      page.goto(agent.homepage_url)
      png = page.screenshot(full_page=False)
      s3.put_object(Bucket="instant-shared",
                    Key=f"thumbnails/{agent.id}.png",
                    Body=png, ContentType="image/png")
  ```

- **Step 3: Mint a presigned URL for the frontend.** 1-hour expiry.

  ```python
  url = s3.generate_presigned_url(
      "get_object",
      Params={"Bucket": "instant-shared", "Key": f"thumbnails/{agent_id}.png"},
      ExpiresIn=3600,
  )
  ```

- **Step 4: Frontend renders directly from MinIO.**

  ```html
  <img src="{{ thumbnail_url }}" loading="lazy" alt="{{ agent.name }}">
  ```

## Why this works on instanode.dev

Marketplace thumbnails are write-once, read-many, and you don't want to pay R2/S3 egress on every card hover. MinIO is S3-API-compatible, so boto3 and any image CDN drop in without code changes. Provisioning is one curl and the bucket starts empty — no IAM policy JSON, no CORS preamble, no per-bucket pricing tier to choose.
