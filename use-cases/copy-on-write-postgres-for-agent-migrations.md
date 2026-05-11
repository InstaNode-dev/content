---
title: Copy-on-write Postgres for agent migrations
category: K. Ephemeral agent runtimes
services: ["pg"]
scenario: An Ardent-style coding agent forks a 6-second copy-on-write clone of prod Postgres, runs the candidate ALTER TABLE, validates row counts, then destroys the clone.
---
