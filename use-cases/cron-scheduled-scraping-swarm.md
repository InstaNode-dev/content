---
title: Cron-scheduled scraping swarm
category: Q. Background/async agent fleets
services: ["mongo", "webhook"]
scenario: A scheduler fans out 5000 cron-triggered scraper agents per hour; each writes its diff to Mongo and pings a webhook only when the page changed.
---
