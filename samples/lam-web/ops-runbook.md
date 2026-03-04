# LAM Web Ops Runbook

Note: This is a writing sample copied from a private repo and included here for portfolio review.

This doc is a lightweight runbook for the lam-web deployment (Django backend + Docusaurus frontend) and its integration with the LAM v2 API.

## Incident class: Console Ops Queue/Metrics failing

The Console “Ops” widgets call the LAM v2 API admin endpoints:

- `GET /v1/admin/enrichment/queue`
- `GET /v1/admin/metrics`

### Quick triage

1) **Bypass the Console and hit the API directly**

Use the public API hostname (not the raw ALB DNS name) and verify:

- `GET https://api.lam-protocol.com/health` returns `200`
- `GET https://api.lam-protocol.com/v1/admin/enrichment/queue` returns `200` with `Authorization: Bearer <admin token>`

If those succeed, the API is healthy and the issue is likely Console configuration.

2) **Confirm Console backend settings**

The lam-web backend uses:

- `LAM_API_URL` (example: `https://api.lam-protocol.com/v1`)
- `LAM_ADMIN_TOKEN` (must match the API’s admin token)

Common failure modes:

- Wrong base URL (missing `/v1`, or pointing at the wrong environment).
- Token mismatch (API returns `401 Invalid admin token`).

### TLS note

If you test using the ALB’s `*.elb.amazonaws.com` DNS name, curl may fail hostname verification (certificate SAN mismatch). Prefer the API hostname that matches the ALB certificate.

## Related docs

- LAM v2 API runbook: `../lam-v2/ops-runbook.md`

### Embeddings / hybrid retrieval runbook

If you’re debugging `/v1/context` hybrid/vector retrieval or passage-embeddings backfills (TEI/OpenAI/self-hosted), follow the LAM v2 docs in this portfolio:

- `../lam-v2/ops-runbook.md` (hybrid/vector context enablement and backfill gotchas)

Operational gotcha we hit:

- Console-created tenants may have a non-empty `scope_user` (often the owner email). Backfill scripts default to empty scope strings, so a backfill can report `processed=0` until you pass `--scope-user` to match the stored passages.
