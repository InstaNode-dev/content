---
title: Cron-scheduled scraping swarm
category: Q. Background/async agent fleets
services: ["mongo", "webhook"]
scenario: A scheduler fans out 5000 cron-triggered scraper agents per hour; each writes its diff to Mongo and pings a webhook only when the page changed.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a scheduler fans out 5000 cron-triggered scraper agents per hour; each writes its diff to Mongo and pings a webhook only when the page changed.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (MongoDB + webhook receiver) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
Set up a 5000-page-per-hour scrape swarm. For each URL, claim a MongoDB on instanode.dev (or reuse) and a webhook. Each scraper agent hashes the page body; if the hash differs from the last run, write the new doc to Mongo and POST to the webhook. If unchanged, skip silently.
```

## Steps to follow

- **Step 1: Provision Mongo + webhook.** One Mongo for diffs, one webhook for change notifications.

  ```bash
  MONGO=$(curl -sX POST https://api.instanode.dev/nosql/new | jq -r .connection_url)
  WH=$(curl -sX POST https://api.instanode.dev/webhook/new | jq -r .receive_url)
  ```

- **Step 2: Schedule the swarm.** Cron triggers a fan-out lambda that dispatches 5000 scraper invocations.

  ```yaml
  # crontab
  0 * * * * /usr/local/bin/fan-out-scrapers --concurrency 200
  ```

- **Step 3: Each scraper diffs and posts.** Idempotent: hash-based.

  ```python
  body = httpx.get(url).text
  h = hashlib.sha256(body.encode()).hexdigest()
  prior = mongo.scrapes.find_one({"url": url}, sort=[("ts", -1)])
  if not prior or prior["hash"] != h:
      mongo.scrapes.insert_one({"url": url, "hash": h, "body": body, "ts": time.time()})
      httpx.post(WEBHOOK_URL, json={"url": url, "changed": True})
  ```

- **Step 4: Read the change feed.** The webhook URL exposes a GET endpoint listing recent posts.

  ```bash
  curl https://api.instanode.dev/api/v1/webhooks/$TOKEN/requests | jq '.[] | .body.url'
  ```

## Why this works on instanode.dev

Webhook receivers are real public HTTPS endpoints — no ngrok, no local tunnel — so 5000 lambdas can POST to the same URL with zero networking setup. Mongo handles unbounded growth in the scrapes collection without schema migrations. Both resources cost $0 on the anonymous tier for the first 24h while you tune the swarm, then convert to hobby tier with one /claim call.

## Related cases

- [Browser job queue with retries](/use-cases/browser-job-queue-with-retries.md) — execution primitive each cron tick fans out onto
- [Scraped product-price history](/use-cases/scraped-product-price-history.md) — longitudinal storage for what these scrapers extract
- [Overnight dossier fleet](/use-cases/overnight-dossier-fleet.md) — another scheduled background fleet pattern with Mongo+S3-compatible storage outputs
