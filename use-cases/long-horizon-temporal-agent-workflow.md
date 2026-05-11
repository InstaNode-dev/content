---
title: Long-horizon Temporal agent workflow
category: Q. Background/async agent fleets
services: ["pg", "nats"]
scenario: A Temporal-backed agent runs a 14-day procurement negotiation across vendors; each step persists in Postgres with NATS signals so a human can interject without breaking the workflow.
---
