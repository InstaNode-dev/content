---
title: Strict-discipline shipping — change → live test → PR → merge
date: 2026-05-11
author: instanode.dev
excerpt: A retrospective on shipping 16 friction fixes in a single session, including the one where unit tests passed but the live cluster told a different story.
---

# Strict-discipline shipping

Most teams ship code by writing it, running the unit tests, opening the PR,
and merging on green CI. We tried something stricter for the last sprint:

1. Write the change
2. Build a container image and roll it out to the live cluster
3. Run real curls against the live cluster — assert the new behavior end-to-end
4. THEN open the PR and merge

It is more work. It found bugs that unit tests missed. Three examples:

## The S3-compatible storage build context

We tried to lift the kaniko build-context cap past 1 MiB by switching from k8s
Secrets to s3:// URLs. Unit tests passed. The Job spec contained
`--context=s3://...` exactly as expected.

Live cluster, the kaniko Job failed. AWS SDK v2 (which kaniko v1.23 ships with)
resolves S3 endpoints in vhost style by default. Our `S3_FORCE_PATH_STYLE`
env var was an SDK v1 knob that the new SDK silently ignored. The bucket name
"instant-build-contexts" was being resolved as a non-existent DNS subdomain.

We never would have caught this from `go test`. The live verification
demanded an init-container that curl-fetches from a presigned URL — kaniko sees
only a local tar file.

## The env validator that wasn't

`POST /db/new?env=Prod` (uppercase, illegal) returned 201 with
`env="production"` in the response. The validation regex was correct. The
helper function returned an error. Two unit tests asserted the regex.

The handler still bypassed it.

`respondError` was calling `c.Status(status).JSON(...)` which returns nil
on a successful body write. The caller's `if err != nil` check was
therefore false on the happy path. Execution continued past the gate and
provisioning succeeded with an empty env, defaulted by NormalizeEnv to
"production".

Centralized fix: `respondError` now returns an `ErrResponseWritten`
sentinel; the ErrorHandlers detect it and avoid overwriting. Twenty-plus
multi-return validators got their teeth back in one commit.

## What we keep

After the sprint we kept the discipline. The friction is real — we spent
extra minutes per PR rolling out builds and running curls. We also stopped
shipping silent bugs.
