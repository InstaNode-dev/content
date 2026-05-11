---
title: Geo-sharded chat-agent fleet
category: R. Edge agents & federated swarms
services: ["redis", "pg"]
scenario: Region-pinned agents in EU, US, and APAC each own a local Redis for session state, replicating only consented memories to a central Postgres for cross-region handoff.
---
