---
title: Anthropic parallel tool_use batch
category: M. Parallel tool execution
services: ["nats", "redis"]
scenario: A Claude turn emits 8 simultaneous tool_use blocks; an executor fans them out to 8 NATS subjects, each handler writes its result to Redis keyed by tool_use_id, and the assistant collects them for the next turn.
---
