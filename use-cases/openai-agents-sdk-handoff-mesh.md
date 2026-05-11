---
title: OpenAI Agents SDK handoff mesh
category: J. Agent swarms & fan-out
services: ["pg", "redis"]
scenario: A triage agent hands off to specialist agents (refund, shipping, fraud) running concurrently, each maintaining its own conversation thread row keyed by handoff_id.
---
