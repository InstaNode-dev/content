---
title: Token-cost ledger per sub-agent
category: N. Multi-agent observability
services: ["webhook", "pg", "redis"]
scenario: Every LLM call from any agent in a fleet emits a webhook with input/output tokens; a ledger agent aggregates spend per tenant in Postgres and triggers throttling at thresholds.
---
