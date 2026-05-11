---
title: Browser job queue with retries
category: E. Browser & automation agents
services: ["nats"]
scenario: A fleet of Playwright workers pulls navigation tasks from a queue, marks them done, and retries on captcha.
---

## Sample agent prompt

```
You're running a fleet of 30 Playwright workers. Pull navigation jobs (url, action, retry_count) from a NATS work-queue. On success, ack. On CAPTCHA detection, requeue with retry_count++ and a 30s delay. After 3 retries, route to the dead-letter subject for human review.
```

## Steps to follow

- **Step 1: Provision NATS JetStream queue.**

  ```bash
  curl -sX POST https://api.instanode.dev/queue/new \
    -H "Authorization: Bearer $T" | jq .
  # response contains connection_url with creds and a JetStream stream name
  ```

- **Step 2: Create the work stream + DLQ subject.**

  ```python
  import nats
  nc = await nats.connect(NATS_URL)
  js = nc.jetstream()
  await js.add_stream(name="browser_jobs",
                      subjects=["jobs.navigate","jobs.dlq"],
                      retention="workqueue")
  ```

- **Step 3: Worker — pull, run, ack or requeue.**

  ```python
  sub = await js.pull_subscribe("jobs.navigate", durable="worker-fleet")
  while True:
      msgs = await sub.fetch(1, timeout=5)
      for m in msgs:
          job = json.loads(m.data)
          try:
              await playwright_run(job["url"], job["action"])
              await m.ack()
          except CaptchaDetected:
              job["retry_count"] += 1
              subject = "jobs.dlq" if job["retry_count"] >= 3 else "jobs.navigate"
              await js.publish(subject, json.dumps(job).encode())
              await m.ack()
  ```

- **Step 4: Drain the DLQ periodically.**

  ```bash
  nats stream view browser_jobs --subjects=jobs.dlq
  ```

## Why this works on instanode.dev

JetStream gives you real durable queues with per-consumer cursors — exactly-once-ish semantics for browser jobs that take 30s each. Compared to SQS, no visibility-timeout tuning hell; compared to Sidekiq, no Redis-cluster setup. One curl, one stream, 30 workers pulling concurrently with zero coordination code.
