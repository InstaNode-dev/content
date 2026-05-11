---
title: Browser-agent action pool
category: M. Parallel tool execution
services: ["mongo", "nats"]
scenario: A Skyvern-style planner emits 20 actions (fill, click, scroll) that fan out across 20 Browserbase tabs simultaneously; results aggregate in Mongo keyed by step_id.
---
