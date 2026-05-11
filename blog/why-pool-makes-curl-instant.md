---
title: How /db/new dropped from 17s to under a second
date: 2026-05-10
author: instanode.dev
excerpt: Pre-warming dedicated Postgres pods, dropping the PVC for the anonymous tier, and caching base images on every node. Three small moves, one big speed-up.
---

# How /db/new dropped from 17s to under a second

For an AI agent making a curl call, the difference between a 200ms response
and a 17s response is the difference between "I'll just use this" and "I'll
write my own".

Anonymous Postgres provisioning used to take 17 seconds end-to-end. Today it
takes about 950 milliseconds. Three changes, in order of impact:

## 1. Warm pool

Every 30 seconds, a background manager checks the `pool_items` table and
makes sure there are at least N ready dedicated Postgres pods (N is
configurable). When `/db/new` is called, it pulls a token off the pool,
renames the namespace's customer-pointing user, and returns the connection URL.
The actual `pg_init`, namespace creation, and pod startup happened minutes
ago.

That's the difference between cooking when an order arrives and pre-cooking.
The pool item swap is one Redis CAS plus one SQL UPDATE. Sub-second.

## 2. Drop the PVC for anonymous

Anonymous data is 24-hour TTL by definition. Persistent storage adds 5–10s of
DOKS block-storage attach time on cold provision. Anonymous-tier pods now use
`emptyDir` — the data lives in pod-local tmpfs and disappears on restart.
That's fine because anyone who wants their data to survive is going to claim
the resource anyway.

This change alone removed about 8 seconds from a cold-path provision.

## 3. Image-puller DaemonSet

The kaniko / Postgres / Redis / Mongo / NATS images were being pulled by every
node on first use. That's another 5–15 seconds gone the first time a node
provisions a specific service type.

A tiny DaemonSet runs an init-container per image that does `/bin/true` after
the image is pulled — the pull itself warms containerd's cache. The DaemonSet
runs on every node, including future ones added to the pool. New nodes are
warm before any customer hits them.

## What we learned

Most of our latency was infrastructure ceremony, not the work the customer
actually cares about. The pool decoupled customer time from provisioning time
entirely. The emptyDir change traded durability we didn't need for speed we
did. The image puller eliminated a one-time-per-node tax that hit at the worst
possible moment.

Combined, they make the platform feel instant. That is the entire pitch.
