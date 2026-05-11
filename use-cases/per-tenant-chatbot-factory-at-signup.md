---
title: Per-tenant chatbot factory at signup
category: L. Agent-factory / spawning patterns
services: ["mongo", "redis", "deploy"]
scenario: When a B2B SaaS user clicks 'Add AI assistant', the factory provisions a dedicated Mongo for that tenant's chat history, a Redis for session state, and deploys an isolated agent container.
---
