---
title: Restate-style durable sidecar
category: Q. Background/async agent fleets
services: ["pg"]
scenario: A Restate sidecar intercepts agent HTTP tool calls and journals them to Postgres so any crash mid-run resumes deterministically without re-charging external APIs.
---
