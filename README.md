# Tucker Olmstead — Backend Technical Writing Portfolio

TL;DR: I build backend systems, then write the docs I wish existed—clear structure, practical examples, trade-offs, failure modes, and copy/paste verification steps.

This repo is my public portfolio for backend-focused technical writing (blog-style) plus case studies of the systems I’ve built and documented.

## Start here (writing samples)

**Original long-form articles (in this repo)**
- Scope-locked APIs (design + failure modes): `writing/scope-locked-apis.md`
- Stripe order recovery (retries/refunds/idempotency): `writing/stripe-order-recovery.md`

**Documentation writing (public repos)**
- LAM Conformance Spec (normative): https://github.com/tuckerjensendev/lam-v2/blob/main/conformance/spec.md
- LAM v2 Ops Runbook: https://github.com/tuckerjensendev/lam-v2/blob/main/OPS_RUNBOOK.md
- LAM Web AWS CI/CD (GitHub Actions → ECS/S3): https://github.com/tuckerjensendev/lam-web/blob/main/CICD_AWS.md
- Stripe Order Recovery System (retry/refund/restore): https://github.com/tuckerjensendev/LinqCV/blob/main/ORDER_RECOVERY_README.md
- Systems/Security Review Packet (bounded, reproducible, falsifiable): https://github.com/tuckerjensendev/lam-v2/blob/main/CSAIL_REVIEW_PACKET.md

## Case studies (projects)
- LAM v2 (TypeScript/Fastify + Postgres): `projects/lam-v2.md`
- LAM Web (Django + Docusaurus): `projects/lam-web.md`
- Orchard (TypeScript/Fastify): `projects/orchard.md`
- LinqCV (Django/DRF + Stripe + React): `projects/linqcv.md`
- Noctara (Django + Vite/React + Electron): `projects/noctara.md`
- LAM “Prove It” 5-minute demo (Docker Compose): `projects/lam-prove-it.md`

## How I write (quick)
- I don’t “handwave” backend claims. If it’s not verifiable from source, I label it as a hypothesis or I cut it.
- I prefer primary sources (official docs, code, RFCs) and I include reproduction steps when possible.
- I optimize for senior readers: assume baseline knowledge; spend time on trade-offs and sharp edges.

## Contact
- Email: tucker.olmstead@yahoo.com
- GitHub: https://github.com/tuckerjensendev
