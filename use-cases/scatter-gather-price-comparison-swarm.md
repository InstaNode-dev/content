---
title: Scatter-gather price comparison swarm
category: J. Agent swarms & fan-out
services: ["nats", "redis"]
scenario: A consumer agent broadcasts a product query to 40 retailer-specific shopper agents over a NATS subject; the first 5 cheapest replies win and the rest are cancelled mid-flight.
---
