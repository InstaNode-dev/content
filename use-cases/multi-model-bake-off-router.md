---
title: Multi-model bake-off router
category: M. Parallel tool execution
services: ["redis", "nats"]
scenario: A router agent dispatches the same prompt to GPT, Claude, and Gemini concurrently; the first valid JSON response wins, and loser turns get cached in Redis for next time.
---
