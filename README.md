# cp-deploy

A general-purpose deployment framework for Linux-based EC2 instances. It uses GitHub Actions, AWS Systems Manager (SSM), and reusable provisioning/deployment scripts to orchestrate remote deployments without running them directly on GitHub runners.

## Overview

- **Remote orchestration** – Deployments are triggered from GitHub Actions but executed on the target host via AWS SSM.
- **Atomic, rollbackable releases** – Each deployment creates an isolated release (worktrees/environments), switches over atomically on success, and automatically rolls back on failure.
- **Host provisioning** – Reusable scripts to install and configure dependencies on target hosts.
- **Separation of concerns** – Secrets/config come from AWS SSM Parameter Store; `TARGET_DIR` (automation checkout) and `DEPLOY_DIR` (live app) are kept distinct.
- **Extensible by design** – Structure supports multiple applications and workflows as needs evolve.

## Current Usage

At the moment, this repo deploys [Superdesk](https://www.superdesk.org/) applications for Canadian Press as concrete examples of these patterns.

| Application | Host Setup | Deployment Script | Runbook | Default Deploy Directory |
|-------------|------------|-------------------|---------|---------------------------|
| CMS | `deploy/hosts/setup-cms-system.sh` | `deploy/services/cms.sh` | [`docs/deploy-cms.md`](docs/deploy-cms.md) | `/var/www/app/cms` |
| Newsroom | `deploy/hosts/setup-newsroom-system.sh` | `deploy/services/newsroom.sh` | [`docs/deploy-newsroom.md`](docs/deploy-newsroom.md) | `/var/www/app/newsroom` |

## Repository Layout

```
cp-deploy/
├── .github/workflows/    # GitHub Actions workflows
├── deploy/
│   ├── deploy.env        # Shared configuration (versions, flags, repo refs)
│   ├── docker-compose.yml # Local dev dependencies (Redis, MongoDB, Elasticsearch)
│   ├── hosts/            # Host provisioning scripts (run with sudo)
│   └── services/         # Per-application deployment scripts (run as deploy user)
└── docs/                 # Runbooks and operational documentation
```

## How It Works

1. **Trigger** – Push to `test-ci-cd` or merged PR targeting `test-ci-cd` (workflow is extensible for other triggers/apps).
2. **Authenticate** – GitHub Actions assumes an AWS IAM role via OIDC.
3. **Remote execution** – `AWS-RunShellScript` is sent to the EC2 instance via SSM.
4. **Checkout** – Repo is cloned/updated to the configured ref in `TARGET_DIR`.
5. **Configuration** – Secrets/env are pulled from AWS SSM Parameter Store into `deploy/deploy.env` with restricted permissions.
6. **Provision** – Host setup script runs with `sudo`.
7. **Deploy** – App script runs as the deploy user (create release, build, switch `current` symlink, restart services, health checks).
8. **Poll & verify** – Workflow polls SSM until completion/timeout; failures stop the job.

## Key Patterns & Features

- **Atomic releases** – Isolated under `releases/` with own deps; promoted to `current` only on success.
- **Automatic rollback** – On activation/health check failure, previous release and Nginx config restored and verified.
- **Release retention** – Failed candidates removed; current + 2 previous releases retained.
- **Health checks** – Optional `APP_HEALTH_URL` for post-deploy readiness.
- **Concurrency control** – Host-wide lock prevents overlapping deployments.
- **Externalized secrets** – Sensitive values from SSM Parameter Store, not committed.
- **Flexible paths** – `TARGET_DIR` and `DEPLOY_DIR` are distinct.

## Extending This Repo

1. **Host provisioning** – Add script in `deploy/hosts/` for target OS/dependencies.
2. **App deployment** – Add script in `deploy/services/` following the release pattern (isolate, build, switch symlink, restart, health check, rollback).
3. **Configure** – Add vars to `deploy.env` and store secrets in SSM Parameter Store.
4. **Orchestrate** – Add/extend GitHub Actions workflow to call via SSM.
5. **Document** – Add runbook under `docs/`.

## Local Development

```bash
cd deploy
docker compose up -d
```

See `deploy/deploy.env` for configurable versions (Redis, MongoDB, Elasticsearch, etc.).

## Important Notes

- **OS target** – Host setup scripts currently target Ubuntu 22.04; OS upgrades are not performed during app deployments.
- **Migrations** – Database migrations are not auto-rolled back. Keep them backwards compatible or have a separate backup/recovery strategy.
- **Rollback caveats** – Current workflow assumes a remote branch; rolling back to tags/commits may require checkout logic changes.
- **General purpose** – This is a general deployment framework; future workflows/apps may be added beyond Superdesk.
