---
title: Inbox-zero agent fleet
category: Q. Background/async agent fleets
services: ["minio", "pg", "webhook"]
scenario: A Cloudflare-Email-style fleet runs one durable agent per user that triages incoming email, files attachments in MinIO, and stores extracted action items in Postgres.
---
