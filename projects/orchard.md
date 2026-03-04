# Orchard — Sandboxed Learning Loop (Fastify)

Repo: https://github.com/tuckerjensendev/orchard

## What it is
Orchard is a minimal, testable learning loop running inside a sandboxed “codebase-world”. It’s an API server that runs bounded episodes (repo fixes, OS fixes, web-research) and stores the artifacts as evidence.

## My role
- Built the API server and the “world” contracts, and documented the architecture and safety boundaries.

## Stack (high level)
- TypeScript / Node.js
- Fastify
- Filesystem-based evidence store + tamper-evident ledger

## Why it’s relevant to backend writing
This repo is basically a playground for: quotas, isolation boundaries, auditability, “safe by default” behaviors, and operational sharp edges. Those are the same topics senior readers care about in real backend systems.

## What I shipped (high level)
- A Fastify server with endpoints that run bounded tasks (episodes) and store tool outputs as content-addressed evidence.
- A thin, tamper-evident ledger for episode/event logging (hash chain).
- Quota’d, read-only “web sensor” features designed with SSRF defenses and explicit on/off toggles.
- Retention concepts (“what delete should mean”) implemented as actual routines, not handwaving.

## Best places to read (docs)
- README (architecture + world contracts): https://github.com/tuckerjensendev/orchard/blob/main/README.md

## Backend topics I can write about from this work
- Evidence stores: content addressing, integrity, and traceability
- Quotas and abuse resistance: bounded tools, request/byte limits, timeouts
- SSRF defenses in “fetch a URL” features
- Retention and deletion semantics (what “delete” should actually mean)

## If you want blog post ideas I can write from this (examples)
- “SSRF defenses that hold up in practice: a checklist for URL fetch features”
- “Evidence stores and audit logs: making artifacts tamper-evident without overengineering”
