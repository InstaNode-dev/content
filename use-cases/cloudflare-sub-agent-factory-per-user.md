---
title: Cloudflare sub-agent factory per user
category: L. Agent-factory / spawning patterns
services: ["pg", "deploy"]
scenario: A Durable-Object-style root agent mints a fresh sub-agent for each end-user, each with its own SQL-backed memory and a deploy slot for that user's tools.
---
