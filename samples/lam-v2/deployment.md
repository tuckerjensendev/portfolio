# LAM Deployment Guide (Cloud / Enterprise / Gov)

Note: This is a writing sample copied from a private repo and included here for portfolio review.

LAM ships as a single **scope-locked HTTP API** (`/v1/*`). All deployments run the same protocol.

The difference between “Cloud”, “Enterprise”, and “Gov” is **who operates the service** and **where memory data lives**.

---

## A) LAM Cloud (SaaS)

**Who runs it:** you  
**Who stores memory:** you (encrypted at rest, keys under your control)

Typical architecture:

- Public HTTPS endpoint (reverse proxy / load balancer)
- LAM API service (this repo)
- Postgres (cells metadata + graph + per-tenant secrets + API keys)
- Cell blob store (filesystem, Postgres, or object storage)

Operational notes:

- Token auth is based on a random token stored as a `sha256(token)` hash in Postgres (`api_keys.token_hash`).
- For early SaaS, you can mint tokens via scripts, or enable the built-in admin endpoints (see below).
- For production, use a **versioned keyring** and rotate via `LAM_MASTER_KEY_ACTIVE_VERSION`:
  - Recommended (cloud): `LAM_MASTER_KEY_PROVIDER=aws-kms` + `LAM_MASTER_KEYS_KMS_B64` (ciphertexts)
  - Allowed (dev/air-gap): `LAM_MASTER_KEYS_B64` (plaintext env keyring)

### Cloud operator endpoints (built-in)

If you set `LAM_ADMIN_TOKEN`, the server enables `/v1/admin/*` routes for:

- Tenant onboarding (`POST /v1/admin/tenants`)
- API key minting (`POST /v1/admin/keys`) - returns the plaintext token once
- Plan management (`POST /v1/admin/plans`, `GET /v1/admin/plans`)
- Tenant plan assignment / suspension (`POST /v1/admin/tenants/:tenantId/plan`, `POST /v1/admin/tenants/:tenantId/suspend`)
- Usage summaries for billing (`GET /v1/admin/usage`)
- Invoice-ready monthly export (`GET /v1/admin/usage/export?month=YYYY-MM&format=csv`)

If `LAM_ADMIN_TOKEN` is unset, these endpoints return `404` (disabled by default).

Tip: set `LAM_DEFAULT_TENANT_PLAN_CODE` to auto-assign a plan when creating tenants via the admin endpoint.

Security tip: for Cloud deployments, do not expose `/v1/admin/*` on the public internet. Prefer network-level isolation
(private service/VPN/IP allowlist). You can also set `LAM_ADMIN_IP_ALLOWLIST` (comma-separated IPs/CIDRs) for an extra
application-layer guard.

---

## Runbook: Console Ops Queue/Metrics failures (Postgres TLS)

This section is for the production incident class where the Console “Ops” page (Queue/Metrics) returns a 500 and the API logs show:

- `SELF_SIGNED_CERT_IN_CHAIN`
- `self-signed certificate in certificate chain`

In our case, the failure was thrown by `pg-pool` while computing queue stats (`getEnrichmentQueueStats`).

### 1) Confirm the *new* code is actually running

If you recently deployed but the error persists, first verify the ECS service is running the expected task definition revision (and that tasks aren’t crash-looping).

From any terminal with AWS CLI access (CloudShell or your local terminal with AWS SSO):

```bash
export AWS_PAGER=""
AWS_REGION=us-east-1
CLUSTER=lam2-cluster
SERVICE=lam2-api-svc

aws --no-cli-pager ecs describe-services --region "$AWS_REGION" --cluster "$CLUSTER" --services "$SERVICE" \
  --query 'services[0].{taskDefinition:taskDefinition,desired:desiredCount,running:runningCount,pending:pendingCount,deployments:deployments[].{status:status,td:taskDefinition,failed:failedTasks,rollout:rolloutState}}' \
  --output json

aws --no-cli-pager ecs list-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --service-name "$SERVICE" \
  --desired-status RUNNING --query 'taskArns' --output text \
| xargs aws --no-cli-pager ecs describe-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --tasks \
  --query 'tasks[].{taskArn:taskArn,taskDefinitionArn:taskDefinitionArn,lastStatus:lastStatus,startedAt:startedAt}' \
  --output table
```

If tasks are stopping immediately (e.g. exit `255`) check the most recent stopped task:

```bash
aws --no-cli-pager ecs list-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --service-name "$SERVICE" \
  --desired-status STOPPED --max-items 5 --query 'taskArns' --output text \
| xargs aws --no-cli-pager ecs describe-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --tasks \
  --query 'tasks[].{taskArn:taskArn,taskDefinitionArn:taskDefinitionArn,stoppedReason:stoppedReason,containers:containers[].{name:name,exitCode:exitCode,reason:reason}}' \
  --output json
```

### 2) ECS architecture mismatch (tasks won’t start)

If deployments fail to start and you see repeated stopped tasks (or a circuit-breaker rollback), verify the task definition `runtimePlatform` matches the image architecture.

```bash
aws --no-cli-pager ecs describe-task-definition --region "$AWS_REGION" --task-definition "$(aws --no-cli-pager ecs describe-services --region "$AWS_REGION" --cluster "$CLUSTER" --services "$SERVICE" --query 'services[0].taskDefinition' --output text)" \
  --query 'taskDefinition.runtimePlatform' --output json
```

We hit a case where `runtimePlatform.cpuArchitecture=ARM64` but the pushed image was effectively `amd64`-only, causing instant failures.
Mitigations:

- Prefer pushing a multi-arch image (`linux/amd64` + `linux/arm64`).
- Or temporarily switch the service to `X86_64` to restore availability while CI is fixed.

### 3) CA bundle: don’t rely on a missing file path

If Node logs warnings like:

- `Ignoring extra certs from /etc/ssl/certs/aws-rds-global-bundle.pem ... No such file or directory`

then `NODE_EXTRA_CA_CERTS` is pointing at a path that does not exist inside the container, so Node will ignore it.

Fix pattern:

- Bake the CA bundle into the image (e.g. under `/app/certs/aws-rds-global-bundle.pem`).
- Set `NODE_EXTRA_CA_CERTS=/app/certs/aws-rds-global-bundle.pem`.
- For Postgres, explicitly configure `pg` SSL with `ssl: { rejectUnauthorized: true, ca: <bundle> }` (via `LAM_PG_SSL_CA_FILE` or `LAM_PG_SSL_CA_PEM_B64`).

### 4) Verify the admin endpoints directly (bypasses the Console)

The Console Ops page reads from API admin endpoints:

- `GET /v1/admin/enrichment/queue`
- `GET /v1/admin/metrics`

If your ALB redirects HTTP → HTTPS, and the TLS cert is for a custom domain, use the custom hostname (not the raw `*.elb.amazonaws.com` DNS) for testing.

Example:

```bash
HOSTNAME=api.lam-protocol.com

curl -i --max-time 10 "https://$HOSTNAME/health"

# Admin endpoints require an admin token
curl -i --max-time 15 -H "Authorization: Bearer $LAM_ADMIN_TOKEN" "https://$HOSTNAME/v1/admin/enrichment/queue"
curl -i --max-time 15 -H "Authorization: Bearer $LAM_ADMIN_TOKEN" "https://$HOSTNAME/v1/admin/metrics" | head -n 30
```

If these succeed (200), but the Console still fails, the issue is likely Console configuration (wrong API URL/token) rather than DB TLS.

---

## B) LAM Enterprise (self-host)

**Who runs it:** the customer (inside their VPC/on‑prem)  
**Who stores memory:** the customer

You typically ship:

- A Docker image (built from this repo)
- A Postgres schema + migrations (included)
- Integration docs and SDKs (this repo)

Quick reference (local):

```bash
# From /lam
docker compose --profile lam up -d
```

---

## C) LAM Gov (self-host / air-gapped)

**Who runs it:** the customer (offline)  
**Who stores memory:** the customer (offline)

Runtime characteristics:

- LAM does not require outbound network access unless you configure a cloud cell store.
- For air-gapped deployments, prefer:
  - `LAM_CELL_STORE=fs` (local disk)
  - `LAM_CELL_STORE=postgres` (DB-backed blobs)

### Air-gap packaging workflow (Docker)

In an internet-connected build environment:

```bash
# Build the LAM image
docker build -t lam:0.1.0 .

# Also pull the Postgres image you plan to use
docker pull postgres:16

# Export images to tar files for transfer
docker save -o lam_0.1.0.tar lam:0.1.0
docker save -o postgres_16.tar postgres:16
```

Transfer `lam_0.1.0.tar` and `postgres_16.tar` to the air-gapped environment, then:

```bash
docker load -i lam_0.1.0.tar
docker load -i postgres_16.tar
```

Run LAM using your internal compose/manifest (or reuse `docker-compose.yml` with images resolved locally). Ensure you set:

- `DATABASE_URL`
- Master key provider config:
  - `LAM_MASTER_KEY_B64` (or `LAM_MASTER_KEYS_B64`) for env provider (default)
  - OR `LAM_MASTER_KEY_PROVIDER=aws-kms` + `LAM_MASTER_KEYS_KMS_B64` for AWS KMS provider
- `LAM_CELL_STORE` (`fs` or `postgres`)
- `LAM_CELL_DIR` (if `fs`)

---

## Choosing a mode

- If the customer can send memory data to you: **Cloud**
- If the customer needs local control/latency/compliance: **Enterprise**
- If the customer is offline/air‑gapped: **Gov**

All three modes use the same client SDK and the same `/v1/*` API.
