---
title: PR-review bot triggered by webhooks
category: A. AI coding agents
services: ["webhook", "redis"]
scenario: A code-review agent receives PR webhooks from GitHub, posts inline comments, and re-reviews on each push while caching diffs between runs.
---

## Sample agent prompt

```
Wire a GitHub PR-review agent. Provision a webhook to receive pull_request and push events, cache the diff per PR head SHA in Redis (so re-reviews on the same SHA are free), and post inline comments via the GitHub API. Return the receiver URL to paste into GitHub's webhook settings.
```

## Steps to follow

- **Step 1: Provision the receiver and the cache.**

  ```bash
  curl -s -X POST https://api.instanode.dev/webhook/new | tee /tmp/wh.json
  curl -s -X POST https://api.instanode.dev/cache/new
  ```

- **Step 2: Configure the GitHub webhook.**

  ```bash
  URL=$(jq -r .receive_url /tmp/wh.json)
  gh api repos/acme/api/hooks -f name=web -f config[url]="$URL" \
    -f config[content_type]=json -F events[]=pull_request -F events[]=push
  ```

- **Step 3: Poller picks up events and caches diffs.**

  ```python
  for ev in fetch_events(WH_URL):
      pr = ev["pull_request"]
      key = f"diff:{pr['head']['sha']}"
      diff = await redis.get(key)
      if not diff:
          diff = gh.pr_diff(pr["number"]).encode()
          await redis.setex(key, 86400, diff)
      review(pr, diff.decode())
  ```

- **Step 4: Post inline comments.**

  ```python
  for c in comments:
      gh.create_review_comment(pr_number=pr["number"], body=c.body,
                               commit_id=pr["head"]["sha"], path=c.path, line=c.line)
  ```

- **Step 5: Re-reviews on same SHA hit the cache.**

  ```bash
  redis-cli get "diff:abc123"  # already there, no GitHub round-trip
  ```

## Why this works on instanode.dev

The webhook receiver shields you from GitHub's retry storms when a deploy fails — events queue up rather than getting dropped. The Redis diff cache keeps repeat reviews on the same SHA cheap, which matters when a busy PR gets 5 force-pushes in an hour.
