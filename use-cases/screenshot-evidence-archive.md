---
title: Screenshot evidence archive
category: E. Browser & automation agents
services: ["minio"]
scenario: A QA agent captures before/after screenshots on every test run and stores them keyed by run-id for diff review.
---

## Sample agent prompt

```
Every Playwright test in this suite should snapshot before-action and after-action screenshots, then upload both to a MinIO bucket provisioned via instanode.dev. Key the objects as runs/<run_id>/<test_name>/<step>.png. On test failure, generate a diff image and store it alongside.
```

## Steps to follow

- **Step 1: Provision the bucket.**

  ```bash
  curl -X POST https://api.instanode.dev/storage/new | tee s.json
  export S3_ENDPOINT=$(jq -r .endpoint s.json)
  export S3_BUCKET=$(jq -r .bucket s.json)
  export AWS_ACCESS_KEY_ID=$(jq -r .access_key s.json)
  export AWS_SECRET_ACCESS_KEY=$(jq -r .secret_key s.json)
  ```

- **Step 2: Snapshot hook in Playwright.**

  ```typescript
  test.afterEach(async ({ page }, info) => {
    const buf = await page.screenshot({ fullPage: true });
    const key = `runs/${RUN_ID}/${info.title}/after.png`;
    await s3.send(new PutObjectCommand({ Bucket: S3_BUCKET, Key: key, Body: buf }));
  });
  ```

- **Step 3: On failure, generate a diff** with pixelmatch.

  ```typescript
  if (info.status !== info.expectedStatus) {
    const diff = pixelmatch(before, after, ...);
    await s3.send(new PutObjectCommand({ Bucket: S3_BUCKET, Key: `runs/${RUN_ID}/${info.title}/diff.png`, Body: diff }));
  }
  ```

- **Step 4: List a run's evidence.**

  ```bash
  aws s3 ls "s3://$S3_BUCKET/runs/$RUN_ID/" --recursive \
    --endpoint-url "$S3_ENDPOINT"
  ```

- **Step 5: Generate a signed URL** for the diff image in the failure report.

## Why this works on instanode.dev

MinIO speaks the S3 API so every Playwright-Python-Go-Rust S3 client just works. `/storage/new` mints a scoped IAM user limited to one bucket — the reviewer's signed URL can't leak into a neighbor's evidence.
