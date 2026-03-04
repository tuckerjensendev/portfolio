# LAM Web AWS CI/CD (GitHub Actions)

Note: This is a writing sample copied from a private repo and included here for portfolio review.

This repo deploys **two independent artifacts**:

- **Backend (Django)** → container image → ECR → ECS service
- **Frontend (Docusaurus)** → static build → S3 → (optional) CloudFront invalidation

Workflows:

- `.github/workflows/deploy-backend-ecs.yml`
- `.github/workflows/deploy-frontend-s3.yml`

## TL;DR (how production updates happen)

- **Backend (Django)**: `git push` to `main`/`master` triggers GitHub Actions to build a Docker image, push it to ECR, and update the ECS service.
	- Trigger is limited to backend changes: the workflow only runs when the push includes paths under `backend/**`.
- **Frontend (Docusaurus)**: `git push` to `main`/`master` triggers GitHub Actions to build the static site and sync it to S3 (then invalidate CloudFront).
	- Trigger is limited to frontend changes: the workflow only runs when the push includes paths under `frontend/**`.

Important distinction:

- Running `git pull` in **AWS CloudShell** only updates the CloudShell VM filesystem.
- ECS does **not** read code from CloudShell.
- To update what’s running in production, rely on the GitHub Actions deploy workflows (or explicitly update ECS).

## Local AWS CLI access (recommended)

You can run all operational checks locally (no CloudShell required) using AWS SSO + AWS CLI v2.

One-time:

```bash
aws configure sso
```

Daily:

```bash
aws sso login --profile lam-admin
export AWS_PROFILE=lam-admin
export AWS_PAGER=""
```

Verification (examples):

```bash
# quick ECS rollout status
aws --no-cli-pager ecs describe-services \
	--cluster lam2-cluster \
	--services lam-web-backend-svc \
	--query 'services[0].{service:serviceName,taskDefinition:taskDefinition,desired:desiredCount,running:runningCount,deployments:deployments[*].{status:status,rollout:rolloutState,td:taskDefinition}}' \
	--output table

# repo-provided status bundle
cd /path/to/lam-web
bash scripts/aws/status.sh
```

## 1) One-time config (in repo)

To keep VS Code’s GitHub Actions linter clean (no “context access might be invalid” warnings), the workflows are configured via **top-level `env:` constants inside the workflow files**.

Edit these:

- `.github/workflows/deploy-backend-ecs.yml`
	- `AWS_REGION`
	- `AWS_ROLE_ARN`
	- `ECR_REPOSITORY`
	- `ECS_CLUSTER`
	- `ECS_SERVICE`
	- `ECS_CONTAINER_NAME`

- `.github/workflows/deploy-frontend-s3.yml`
	- `AWS_REGION`
	- `AWS_ROLE_ARN`
	- `S3_BUCKET`
	- `CLOUDFRONT_DISTRIBUTION_ID` (optional; leave empty to skip invalidation)

## 2) IAM permissions (high level)

The OIDC role needs permission to:

- Push images: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`
- Update ECS: `ecs:DescribeServices`, `ecs:DescribeTaskDefinition`, `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:ListTasks`, `ecs:DescribeTasks`
- Pass roles (commonly required when registering task defs): `iam:PassRole` for the task execution role / task role already referenced by the task definition
- Upload site: `s3:ListBucket`, `s3:PutObject`, `s3:DeleteObject` on the target bucket
- Invalidate: `cloudfront:CreateInvalidation` on the distribution

## 3) Notes

- Backend workflow tags images as `${GITHUB_SHA}` and `latest`.
- ECS deploy logic is **in-place**: it fetches the currently-running task definition, patches the container image, registers a new revision, then updates the service.
- Because `lam-web` and `lam-v2` have separate repos and roles/secrets, `git push` in one repo cannot affect the other unless you intentionally point them at the same AWS resources.
