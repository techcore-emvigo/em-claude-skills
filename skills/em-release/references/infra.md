# Infrastructure — Gap Detection

## Docker Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| `FROM node:latest` or `FROM python:latest` | Unpinned base image tag — non-deterministic builds | 🟠 |
| Running as root | No `USER appuser` instruction — container runs as root | 🟠 |
| Secret in `ENV` instruction | `ENV API_KEY=real_value` in Dockerfile | 🔴 |
| No `.dockerignore` | `.git`, `node_modules`, test files not excluded — large image | 🟡 |
| No `HEALTHCHECK` | Service container with no health check instruction | 🟠 |
| Single-stage build | Build tools and source included in final image | 🟡 |
| No image vulnerability scan | No Trivy / Snyk scan in CI pipeline | 🟠 |

## Kubernetes Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| No `resources.requests` / `limits` | Container with no CPU/memory constraints — noisy neighbour risk | 🟠 |
| Single replica | `replicas: 1` on a production deployment | 🟠 |
| No liveness/readiness probe | Deployment with no health probes | 🟠 |
| `runAsNonRoot: false` | Security context allows root | 🟠 |
| `allowPrivilegeEscalation: true` | Container can escalate to root privileges | 🔴 |
| Secret in `ConfigMap` | Sensitive value stored in ConfigMap instead of Secret | 🔴 |
| `0.0.0.0/0` ingress rule | NetworkPolicy or security group allowing all inbound traffic | 🔴 |
| No `PodDisruptionBudget` | Service can go to zero replicas during node drain | 🟠 |

## Terraform Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Wildcard IAM | `Action: "*"` or `Resource: "*"` in IAM policy | 🔴 |
| Public S3 bucket | `block_public_acls = false` without explicit justification | 🔴 |
| `0.0.0.0/0` ingress | Security group ingress open to all IPs (except ports 80/443 behind ALB) | 🔴 |
| Hardcoded credentials | `access_key`, `secret_key` values in `.tf` files | 🔴 |
| No `prevent_destroy` on stateful resources | RDS, S3, DynamoDB without `lifecycle { prevent_destroy = true }` | 🟠 |
| No remote state backend | State file stored locally — not shared, not locked | 🟠 |
| Unpinned provider version | `version = ">= 5.0"` instead of `~> 5.0` | 🟡 |

## CI/CD Gaps

| Gap | What to Look For | Severity |
|---|---|---|
| Unpinned action version | `uses: actions/checkout@v3` instead of pinned SHA | 🟠 |
| Secret in pipeline YAML | Hardcoded value in `env:` block | 🔴 |
| `apply -auto-approve` in production | Terraform apply without human review | 🔴 |
| No security scan step | Pipeline with no SAST or dependency vulnerability scan | 🟠 |
| No coverage gate | Tests pass but coverage threshold not enforced | 🟡 |
| Deploy from unprotected branch | CD triggered from any branch, not just `main`/`release` | 🟠 |

## Generation Checklist
- [ ] Docker: pinned base tag, non-root user, multi-stage build, HEALTHCHECK
- [ ] K8s: resource limits, 2+ replicas, liveness+readiness probes, non-root security context
- [ ] Terraform: least-privilege IAM, `prevent_destroy` on stateful, remote state, `tfsec` scan
- [ ] CI: actions pinned by SHA, secrets from store, security scan step, coverage gate

---

## Release-Time Infrastructure Checklist

| Gap | What to Look For | Severity |
|---|---|---|
| Terraform scripts not updated for this release | Infra drift; production can't be reproduced from code | 🟠 |
| New cloud services not documented or tracked | Unknown cloud costs and dependencies | 🟠 |
| Cloud service costs not monitored post-release | Unexpected cost overruns | 🟠 |
| Least privilege not applied to new cloud configs | Over-permissioned services expand blast radius | 🔴 |
| Secrets/credentials not in secrets manager | Credentials in code, env files, or config | 🔴 |
| Infrastructure as code not maintained (manual changes made) | Snowflake environments; can't reproduce | 🟠 |
| Backup plan not in place for new cloud-stored data | Data loss with no recovery path | 🔴 |
| Access to new cloud services not monitored | Unauthorized use undetected | 🟠 |
| Data retention policies not applied to new data stores | Compliance violation | 🟠 |
| Migration scripts for this release not reviewed and tested in staging | Migration fails or corrupts data in production | 🔴 |
