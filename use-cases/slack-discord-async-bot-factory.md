---
title: Slack/Discord async bot factory
category: Q. Background/async agent fleets
services: ["webhook", "pg"]
scenario: A factory spins up one durable agent per Slack workspace that subscribes to events via webhook, batches them, and runs nightly summarizer jobs persisting digests in Postgres.
---
