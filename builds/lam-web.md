# LAM Web: Docs Site + Django Console

## What it is
A marketing/docs site plus a Django backend that acts as an admin + “console” layer on top of the LAM API (tenant provisioning, API key minting UI, usage export, ops visibility).

## My role
- Built the system and wrote the operational + developer documentation (runbooks, CI/CD notes, integration gotchas).

## Stack (high level)
- Frontend: Docusaurus (React)
- Backend: Django (admin + small API)
- Deployment: GitHub Actions → ECS (backend) and S3/CloudFront (frontend)

## What I shipped (high level)
- A docs site + a console backend that integrates with an external API service (LAM) and makes onboarding/operations self-serve.
- Ops-facing troubleshooting docs that start by bypassing the UI and validating the upstream service directly.
- CI/CD docs that explain the real deployment mechanism (what changes trigger which workflow; what CloudShell can’t do; how to verify rollouts).

## Best places to read (docs)
- [Ops runbook (console ↔ API triage)](../docs/lam-web/ops-runbook.md)
- [AWS CI/CD notes (ECS/S3)](../docs/lam-web/aws-cicd.md)

## Backend topics I can write about from this work
- Designing “console” layers: where to keep responsibility boundaries between UI/backend/core API
- CI/CD you can reason about: least-magic deployments and clear rollback paths
- Production debugging playbooks: bypass the UI; test the API; isolate config vs service faults

## If you want blog post ideas I can write from this (examples)
- “Console vs core API: where admin responsibilities should live (and why it matters)”
- “GitHub Actions → ECS deployments: how to verify what’s actually running”
