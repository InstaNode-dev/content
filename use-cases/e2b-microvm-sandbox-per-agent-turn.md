---
title: E2B microVM sandbox per agent turn
category: K. Ephemeral agent runtimes
services: ["pg", "deploy"]
scenario: Each turn of a code-writing agent spins up a fresh E2B-style microVM whose scratch Postgres connection_url lives only for the lifetime of the sandbox, then is reaped.
---
