# Architecture

## The app

TaskBoard API, Python + FastAPI + PostgreSQL.

- `GET /health` — returns 200 and DB connectivity status. Used by ALB,
  Kubernetes liveness/readiness probes, and uptime checks. Never put auth on this.
- `GET /metrics` — Prometheus exposition format. Counts requests, latency
  histogram, in-flight requests. No business data in metric labels.
- `POST /tasks`, `GET /tasks`, `GET /tasks/{id}`, `PUT /tasks/{id}`, `DELETE /tasks/{id}`
  — a task has: id, title, status (`todo` / `doing` / `done`), created_at, updated_at.
- Config only from environment variables. No hardcoded credentials, URLs, or ports
  in code. `DATABASE_URL` comes from Secrets Manager in AWS, from `.env` locally
  (and `.env` is gitignored — see Story 1).

## Environments

| Environment | Where | Purpose |
|---|---|---|
| local | Docker Compose | Fast iteration, tests, Compose Postgres |
| dev | ECS Fargate (Story 6) | First cloud deployment, prove CD |
| prod-like | EKS + Helm (Story 7 onward) | The main operated environment |

One image, all environments. The image built in CI is the image deployed to
ECS and to EKS. Environment differences live in config (env vars, Helm values,
Terraform variables), never in a rebuilt image.

## Data

- PostgreSQL only. No cache, queue, or second database in the core project —
  those are stretch goals after Story 12.
- Migrations: one simple, explicit mechanism (Alembic, or versioned SQL files
  applied at deploy time). Pick one in Story 2 and do not switch mid-project.
- Backups: RDS automated backups on, in Story 5. Prove a restore in Story 12's
  game day only if time allows — it is listed as a stretch there, not a blocker.

## Networking (AWS)

- One VPC, public subnets for the ALB / NAT, private subnets for ECS tasks,
  EKS nodes, and RDS. RDS is never publicly accessible.
- Security groups are least-privilege: ALB accepts 443/80 from the internet,
  app accepts traffic only from the ALB / Ingress, RDS accepts traffic only
  from the app security group.
- State: Terraform remote state in S3 with DynamoDB locking (Story 5).
  Never commit `.tfstate` or `.terraform/` to Git.

## Observability decisions

- Metrics: Prometheus on EKS (Story 10), CloudWatch for ECS (Story 6).
- Logs: structured JSON logs to stdout, collected by CloudWatch / Grafana Loki.
  Log a request id on every line; never log secrets, tokens, or request bodies
  containing credentials.
- Dashboards (Story 10): request rate, error rate, p95 latency, CPU/memory,
  DB connections. One dashboard per environment, not one panel per idea.
- Alerts (Story 12): alert on symptoms (error rate, latency, saturation),
  not on every cause. Every alert must link to a runbook.

## Security decisions

- Secrets in AWS Secrets Manager for the app (Story 5-6); Vault is learned in
  Story 11 as a second mechanism, not as a replacement you must migrate to.
- Images scanned with Trivy in CI (Story 4) and again before deploy.
- Terraform scanned with Checkov (Story 5).
- IAM: CI gets a dedicated deploy role with only the permissions it needs
  (Story 6). No long-lived admin keys in GitHub secrets if OIDC is available —
  prefer GitHub OIDC to AWS.

## Decisions you make once, and record here

When a story forces a choice (Alembic vs SQL migrations, kind vs EKS that week,
Datadog vs New Relic for the optional stretch), write the choice and the reason
in this file in your PR. Future-you, and anyone reviewing the repo, should be
able to see why, not just what.
