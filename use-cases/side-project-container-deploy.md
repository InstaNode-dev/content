---
title: Side-project container deploy
category: H. Indie & SaaS founders
services: ["deploy"]
scenario: An indie hacker ships a Dockerized app to a subdomain with one HTTP call — no DevOps account, no Helm chart, no IaC.
---

## Sample agent prompt

```
I have a Dockerized Go app in this repo. Ship it to a public subdomain via instanode.dev /deploy/new. No need for a Render or Fly account — just give me a URL I can hit from anywhere, with HTTPS, in under 60 seconds.
```

## Steps to follow

- **Step 1: Build and push the image.**

  ```bash
  docker build -t ghcr.io/me/sideproject:v1 .
  docker push ghcr.io/me/sideproject:v1
  ```

- **Step 2: Deploy via one HTTP call.**

  ```bash
  curl -X POST https://api.instanode.dev/deploy/new \
    -H "Content-Type: application/json" \
    -d '{
      "image": "ghcr.io/me/sideproject:v1",
      "port": 8080,
      "subdomain": "my-thing",
      "env": {"PORT": "8080"}
    }'
  ```

- **Step 3: Hit the returned URL.**

  ```bash
  curl https://my-thing.instanode.dev/healthz
  ```

- **Step 4: Tail logs while you debug.**

  ```bash
  curl -N "https://api.instanode.dev/deploy/$TOKEN/logs?follow=1"
  ```

- **Step 5: When you push v2, redeploy** by hitting /deploy/new with the new image tag — old containers drain automatically.

## Why this works on instanode.dev

No cloud account, no DNS records, no TLS cert dance. The platform issues a `*.instanode.dev` subdomain with HTTPS at deploy time. Hobby tier ($9/mo) gives one small app with persistent disk; pro lifts it to five.
