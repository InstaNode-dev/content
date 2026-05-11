---
title: Cursor background-agent worktree
category: K. Ephemeral agent runtimes
services: ["redis", "webhook", "deploy"]
scenario: A Cursor background agent claims an isolated dev environment for each branch it's working on, with its own Redis for build caches and a webhook to ping the IDE when the PR is ready.
---
