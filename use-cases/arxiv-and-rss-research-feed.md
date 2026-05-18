---
title: arXiv-and-RSS research feed
category: D. Personal AI
services: ["webhook", "redis", "storage"]
scenario: A research agent receives webhook pings from arXiv and RSS bridges, dedupes via Redis, and stores PDFs for later retrieval.
---

## Prompt for any LLM (no setup needed)

Paste this into ChatGPT, Claude, or Gemini — no MCP, no API key, no install:

```
Read https://instanode.dev/llms.txt for the API.

I want to: a research agent receives webhook pings from arXiv and RSS bridges, dedupes via Redis, and stores PDFs for later retrieval.

Write a complete runnable script (bash + whatever language fits) that:
- Provisions the services I need (webhook receiver + Redis + S3-compatible storage) from instanode.dev
- Does the work above end-to-end
- Prints expected output at each step
- Tells me how to claim the resources at the end if I want to keep them past 24 hours

Use real curl commands against api.instanode.dev. Quote the actual response shapes from llms.txt.
```

## Sample agent prompt

```
You're my research agent. Subscribe to arXiv + RSS bridges via a webhook URL. On each ping, dedupe by paper_id in Redis (TTL 30 days). For new papers, download the PDF to S3-compatible storage under `papers/{arxiv_id}.pdf`. Surface unread papers when I ask.
```

## Steps to follow

- **Step 1: Provision the trio.**

  ```bash
  HOOK=$(curl -sX POST https://api.instanode.dev/webhook/new -H 'Content-Type: application/json' -d '{"name":"arxiv-and-rss-research-feed-webhook"}' -H "Authorization: Bearer $T" | jq -r .receive_url)
  REDIS=$(curl -sX POST https://api.instanode.dev/cache/new -H 'Content-Type: application/json' -d '{"name":"arxiv-and-rss-research-feed-cache"}' -H "Authorization: Bearer $T" | jq -r .connection_url)
  curl -sX POST https://api.instanode.dev/storage/new -H 'Content-Type: application/json' -d '{"name":"arxiv-and-rss-research-feed-storage"}' -H "Authorization: Bearer $T" > s3.json
  ```

- **Step 2: Register the webhook with your RSS bridge service.** Point arXiv-RSS-bridge (or similar) at `$HOOK`.

- **Step 3: Poll the webhook inbox, dedupe, fetch PDF.**

  ```python
  pings = requests.get(f"https://api.instanode.dev/api/v1/webhooks/{TOKEN}/requests").json()
  for p in pings:
      paper = p["body"]
      if r.set(f"seen:{paper['id']}", "1", nx=True, ex=86400*30):
          pdf = requests.get(paper["pdf_url"]).content
          s3.put_object(Bucket="instant-shared",
                        Key=f"papers/{paper['id']}.pdf", Body=pdf)
          r.lpush("queue:unread", paper["id"])
  ```

- **Step 4: Surface unread on demand.**

  ```python
  unread = [r.lpop("queue:unread") for _ in range(10) if r.llen("queue:unread")]
  ```

## Why this works on instanode.dev

Personal-research agents shouldn't need a server. A webhook URL is the inbox, Redis SET-NX is the dedupe primitive, S3-compatible storage is the PDF archive — three curls, no Lambda, no DynamoDB GSI. The webhook receive endpoint is queryable via HTTP so the agent can pull on its own cadence instead of running a 24/7 listener.

## Related cases

- [Obsidian-vault embedding sync](/use-cases/obsidian-vault-embedding-sync.md) — downstream sink that indexes the papers this feed pulls in
- [Voice-memo capture pipeline](/use-cases/voice-memo-capture-pipeline.md) — another personal-AI ingestion pipeline with audio instead of papers
- [Overnight dossier fleet](/use-cases/overnight-dossier-fleet.md) — consumes the same kind of feed to generate scheduled briefings
