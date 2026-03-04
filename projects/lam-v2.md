# LAM v2 — Lossless Associative Memory (v0.1)

Repo: https://github.com/tuckerjensendev/lam-v2

## What it is
LAM is a scope-locked HTTP “memory sidecar” with proof semantics: retrieved memory items can point back to bounded source excerpts via decode endpoints.

## My role
- Built the reference implementation and wrote the protocol + ops documentation.
- Wrote reviewer-facing material designed to be reproducible and falsifiable (not “trust me”).

## Stack (high level)
- TypeScript / Node.js
- Fastify (HTTP API)
- PostgreSQL (cells metadata + graph + per-tenant secrets)
- Observability hooks: health/ready endpoints and metrics (OTel/Prometheus-style surfaces)

## Why it’s a strong writing sample
This project forced me to write like a reviewer is trying to break it:
- clear normative rules (what MUST happen)
- explicit failure modes (how isolation breaks in real systems)
- “how to verify” steps (curl/AWS CLI)

## What I shipped (high level)
- Scope-locked API surface (`/v1/*`) where scope is derived from auth and the server rejects client-supplied scope fields by default.
- “Proof” paths that let you mechanically verify returned excerpts via decode endpoints (bounded quotes, stable identifiers, deterministic behavior).
- Conformance suite + fixtures to validate core behaviors across independent implementations (different languages/storage backends).
- Deployment + runbooks focused on production reality (rollouts, TLS, image/CPU mismatches, “is the new version actually running?” checks).

## Best places to read (docs)
- Conformance spec (normative): https://github.com/tuckerjensendev/lam-v2/blob/main/conformance/spec.md
- Ops runbook: https://github.com/tuckerjensendev/lam-v2/blob/main/OPS_RUNBOOK.md
- Deployment guide: https://github.com/tuckerjensendev/lam-v2/blob/main/DEPLOYMENT.md
- Systems/security review packet: https://github.com/tuckerjensendev/lam-v2/blob/main/CSAIL_REVIEW_PACKET.md

## Backend topics I can write about from this work
- Scope isolation: “derive scope from auth; reject client-supplied scope”
- Evidence-backed systems: bounded quoting, decode semantics, deterministic behavior
- Postgres in production: migrations, TLS pitfalls, extension availability (pgvector)
- Workers/queues: async enrichment, backoff/retry, status endpoints
- Observability: what to expose (and what not to) in health/metrics/admin endpoints

## If you want blog post ideas I can write from this (examples)
- “Scope-Locked APIs: the boring rules that prevent exciting incidents”
- “Runbooks that work: writing production triage steps that don’t assume tribal knowledge”
- “Evidence-backed systems: how to make ‘citations’ mechanically verifiable”
