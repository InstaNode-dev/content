---
title: LangGraph fan-out research agents
category: J. Agent swarms & fan-out
services: ["nats", "pg"]
scenario: A planner agent splits a research question into 12 sub-queries and dispatches them to parallel LangGraph workers that each hit different sources, returning JSON to a NATS subject the reducer subscribes to.
---
