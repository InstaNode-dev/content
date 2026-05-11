---
title: Live agent-topology graph
category: N. Multi-agent observability
services: ["nats", "pg"]
scenario: Each spawn/handoff event is published to a NATS subject; a topology agent subscribes and maintains a real-time graph of which parent spawned which child in Postgres for the dashboard.
---
