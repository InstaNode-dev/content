---
title: Devin-style PR-bot fleet
category: L. Agent-factory / spawning patterns
services: ["pg", "webhook", "deploy"]
scenario: A Cognition-Devin-style orchestrator spawns one fresh agent worker per inbound GitHub issue, each with its own scratch Postgres for running migrations against, and reports back via webhook.
---
