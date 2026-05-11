---
title: Federated-learning aggregator
category: R. Edge agents & federated swarms
services: ["webhook", "minio", "pg"]
scenario: Edge agents on user devices compute private gradients and POST them to an aggregator webhook; the aggregator stores rounds in MinIO and the new global weights in Postgres.
---
