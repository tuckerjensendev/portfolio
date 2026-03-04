# LAM v2 Ops Runbook

Note: This is a writing sample copied from a private repo and included here for portfolio review. Some identifiers were redacted.

This doc is a lightweight runbook for common production checks for the LAM v2 API service.

## Incident: Console Ops Queue/Metrics failing (Postgres TLS)

**Symptoms**

- Console Ops page shows Queue/Metrics failures.
- API logs show `SELF_SIGNED_CERT_IN_CHAIN` / `self-signed certificate in certificate chain`.
- Stack traces often point at `pg-pool` during queue stats (e.g. `getEnrichmentQueueStats`).

**Common root causes we hit**

1) **New deploy isn’t actually running**

- ECS service stays on an older task definition because newer tasks crash-loop (e.g. exit `255`) and the circuit breaker rolls back.

2) **Task CPU architecture mismatch**

- Task definition uses `runtimePlatform.cpuArchitecture=ARM64` but the pushed image is `linux/amd64` only (or vice versa).
- Result: tasks fail to start (often immediately) and never become RUNNING.

3) **Node is ignoring `NODE_EXTRA_CA_CERTS`**

- Node will ignore the extra cert bundle if the file path doesn’t exist in the container.
- Typical log: `Ignoring extra certs from /etc/ssl/certs/aws-rds-global-bundle.pem ... No such file or directory`.

4) **Admin token mismatch**

- Console Ops calls `/v1/admin/*` endpoints.
- If `LAM_ADMIN_TOKEN` doesn’t match, API returns `401 Invalid admin token`.

### Step 1 - Verify ECS is actually running the new revision

```bash
export AWS_PAGER=""  # disable interactive pager
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

If tasks are failing to start, inspect stopped tasks:

```bash
aws --no-cli-pager ecs list-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --service-name "$SERVICE" \
  --desired-status STOPPED --max-items 5 --query 'taskArns' --output text \
| xargs aws --no-cli-pager ecs describe-tasks --region "$AWS_REGION" --cluster "$CLUSTER" --tasks \
  --query 'tasks[].{taskArn:taskArn,taskDefinitionArn:taskDefinitionArn,stoppedReason:stoppedReason,containers:containers[].{name:name,exitCode:exitCode,reason:reason}}' \
  --output json
```

### Step 2 - Check task architecture vs image

```bash
aws --no-cli-pager ecs describe-task-definition --region "$AWS_REGION" --task-definition "$(aws --no-cli-pager ecs describe-services --region "$AWS_REGION" --cluster "$CLUSTER" --services "$SERVICE" --query 'services[0].taskDefinition' --output text)" \
  --query 'taskDefinition.{runtimePlatform:runtimePlatform,containerImage:containerDefinitions[0].image}' \
  --output json
```

Mitigations:

- Prefer publishing multi-arch images (`linux/amd64` + `linux/arm64`).
- Or temporarily set `runtimePlatform.cpuArchitecture` to match the image you actually have available.

### Step 3 - Verify the endpoints directly (bypass Console)

Use the public API hostname (the TLS certificate usually does **not** match the raw ALB DNS name).

```bash
HOSTNAME=api.lam-protocol.com

curl -i --max-time 10 "https://$HOSTNAME/health"

# Admin endpoints require LAM_ADMIN_TOKEN
curl -i --max-time 15 -H "Authorization: Bearer $LAM_ADMIN_TOKEN" "https://$HOSTNAME/v1/admin/enrichment/queue"
curl -i --max-time 15 -H "Authorization: Bearer $LAM_ADMIN_TOKEN" "https://$HOSTNAME/v1/admin/metrics" | head -n 30
```

If these are `200`, but the Console still fails, it’s likely Console configuration (wrong API URL/token) rather than DB TLS.

### Step 4 - Secrets Manager `valueFrom` parsing gotcha (ECS)

In ECS task definitions, `containerDefinitions[].secrets[].valueFrom` can use an extended form like:

- `arn:aws:secretsmanager:REGION:ACCOUNT:secret:secret-name:JSON_KEY::`

That string is **not** a valid Secrets Manager `secret-id` as-is.

To fetch the secret for manual testing:

```bash
TOKEN_REF='arn:aws:secretsmanager:us-east-1:<AWS_ACCOUNT_ID>:secret:lam2/prod/app-REDACTED:LAM_ADMIN_TOKEN::'

SECRET_ARN="$(echo "$TOKEN_REF" | cut -d: -f1-7)"
JSON_KEY="$(echo "$TOKEN_REF" | cut -d: -f8)"

ADMIN_TOKEN=$(
  aws --no-cli-pager secretsmanager get-secret-value --region us-east-1 \
    --secret-id "$SECRET_ARN" --query 'SecretString' --output text \
  | python3 -c 'import sys,json; key=sys.argv[1]; s=sys.stdin.read(); j=json.loads(s); v=j.get(key); print(v if isinstance(v,str) else "")' "$JSON_KEY" \
  | tr -d '\r\n'
)
```

## Where to look next

- API logs (CloudWatch log group is typically `/ecs/<service-name>`).
- Admin endpoints behavior:
  - `401` usually means token missing/mismatch.
  - `403` often means `LAM_ADMIN_IP_ALLOWLIST` is blocking the caller.
  - `500` means an internal error; check logs for the real stack.

## Feature: Hybrid/vector `/v1/context` by default

`/v1/context` always supports lexical retrieval. For better recall (e.g. singular/plural like `banana` vs `bananas`) you typically want **hybrid lexical+vector** ranking.

Current behavior:

- If embeddings are configured via `LAM_EMBEDDINGS_CMD`, LAM auto-enables vector ranking for `/v1/context`.
- Override knob: `LAM_VECTOR_SEARCH=0|1` forces vector off/on.

### Production enablement (ECS)

1) Ensure pgvector is available in Postgres
- LAM migrations attempt `create extension vector` when the extension exists in `pg_available_extensions`.

2) Configure embeddings in the API task definition

OpenAI quick start (runs inside the `lam-v2` container):

- `LAM_EMBEDDINGS_CMD="node scripts/embeddings/openai-embedder.js"`
- `LAM_EMBEDDINGS_MODEL="text-embedding-3-small"`
- `OPENAI_API_KEY` (store as an ECS secret)

Self-hosted quick start (Hugging Face TEI inside your VPC):

- Run a TEI service (CPU or GPU) behind an internal load balancer / private DNS.
- `LAM_EMBEDDINGS_CMD="node scripts/embeddings/tei-embedder.js"`
- `TEI_BASE_URL="http://<tei-host>:<port>"`
- `LAM_EMBEDDINGS_MODEL="BAAI/bge-small-en-v1.5"` (bookkeeping key; TEI itself is configured with its own model)

3) Backfill passage embeddings for existing data

Run once (per tenant/scope/namespace/passage-kind) in an environment that has DB connectivity and the same embeddings env vars.

If you are running the backfill inside ECS, avoid `npm run ...` scripts that include `npm run build`.
Production images install deps with `--omit=dev`, so `tsc` is not available and the task may exit with code `127`.

Prefer running the already-built `dist/` script directly:

- `node dist/scripts/passage-embeddings-backfill.js --tenant 1 --namespace default --passage-kind sentence_window_v1`

Common gotchas:

- `processed=0` usually means your filters don’t match reality. The backfill filters on `scope_user`, `scope_org`, `scope_project`, `namespace`, and `passage_kind`.
  - Defaults are empty strings for scope + namespace, and `passage_kind=sentence_window_v1`.
  - Console-created tenants may have a non-empty `scope_user` (e.g. the owner email), so pass `--scope-user "you@company.com"`.
- `Embeddings command timed out`: raise `LAM_EMBEDDINGS_TIMEOUT_MS` (LAM wrapper) and `TEI_TIMEOUT_MS` (HTTP client) to a value that fits your model + batch size.
- `fetch failed` / connection timeouts: TEI must be reachable from the API/worker tasks (security groups must allow TCP/8080 to the TEI service).

Without backfill, older passages won’t be retrievable via vector until they are embedded.

## Related docs

- Deployment guidance and deeper context: see `deployment.md`.
