# my-devops-project — TaskBoard API, built the DevOps way

A hands-on DevOps learning project. You build it yourself, one story at a time.
No app code is shipped in this scaffold on purpose — the stories below *are* the build.

## The idea

**TaskBoard API** — a small, real, boring-in-a-good-way service:

- A Python **FastAPI** API for managing tasks: create, list, update, delete.
- Backed by **PostgreSQL**.
- Endpoints: `GET /health`, `GET /metrics`, plus CRUD under `/tasks`.
- No frontend needed to learn DevOps. `curl` is the client. A tiny frontend is a stretch goal, not a requirement.

Why this app, and not something fancier? Because the learning is in the pipeline,
the infrastructure, and the operations — not in the app. A simple CRUD API gives you
a database, migrations, config, secrets, health checks, metrics, and logs to operate,
without app complexity getting in the way.

## Target architecture (end state)

```
Developer ── PR ──► GitHub Actions CI ──► Trivy scan ──► ECR image
                                   │
Terraform (VPC, IAM, ECR, RDS, Secrets Manager, EKS)
                                   │
                    ┌──────────────┴──────────────┐
                 ECS Fargate                 EKS + Helm
              (Story 6, first)          (Story 7, then primary)
                    │                          │
        CloudWatch logs/metrics     Prometheus + Grafana
        Grafana alerts ──► PagerDuty / Opsgenie ──► Runbook
```

You deploy to ECS first because it is simpler and proves the pipeline end to end.
You then move the same image to EKS with Helm, because that is where the
Kubernetes, Helm, Prometheus, and GitOps-style learning lives.

## How to work through it

1. Work in order. Each story depends on the previous one.
2. One story = one branch = one pull request. No direct pushes to `main`.
3. A story is done only when its **Acceptance criteria** pass — not when the code "looks right".
4. If a story is too expensive on AWS, use the local alternative noted in that story
   (Docker Compose / kind / minikube), and say so in the PR.
5. Tear down AWS resources in Story 12. Do not leave EKS/RDS running idle.

Full detail, tasks, and acceptance criteria: [docs/stories.md](docs/stories.md)
Architecture notes and decisions: [docs/architecture.md](docs/architecture.md)
Which tool is learned where: [docs/tools-map.md](docs/tools-map.md)

## The stories

| # | Story | Main tools | Teaches |
|---|-------|------------|---------|
| 1 | Repo foundations & branching | Git, GitHub | Branching, PRs, branch protection |
| 2 | Build TaskBoard API locally | Python, FastAPI, pytest, PostgreSQL | API + tests + local DB |
| 3 | Dockerize it | Docker, Docker Compose | Multi-stage builds, image hygiene |
| 4 | CI with GitHub Actions | GitHub Actions, Trivy | Lint, test, build, scan in CI |
| 5 | Terraform AWS foundations | Terraform, AWS, Checkov | VPC, IAM, ECR, RDS, remote state |
| 6 | Deploy to ECS Fargate | ECS, ALB, CloudWatch, Secrets Manager | First real deployment + CD |
| 7 | Kubernetes on EKS + Helm | EKS, Helm, kubectl | Deployments, Services, Ingress, HPA |
| 8 | Jenkins pipeline (parallel track) | Jenkins | Jenkins vs GitHub Actions, agents |
| 9 | Configuration with Ansible | Ansible | Idempotent config, roles, inventory |
| 10 | Observability | Prometheus, Grafana, CloudWatch, Datadog/New Relic (optional) | Metrics, dashboards, logs, tracing |
| 11 | Secrets & security hardening | Secrets Manager, Vault, Snyk, IAM | Least privilege, rotation, scanning |
| 12 | SLOs, alerting & incident drill | Grafana alerts, PagerDuty/Opsgenie | SLOs, error budgets, runbooks, game day |

## Cost warning

EKS + RDS + ALB cost real money even when idle. Use the smallest sizes in the
stories, work in one region, and run the teardown in Story 12 when you pause.
Story 7 has a kind/minikube alternative if you want the Kubernetes learning
without the EKS bill that week.

## Repo layout (as you build it)

```
app/                    # FastAPI app (Story 2)
  tests/                # pytest (Story 2)
infra/
  terraform/            # AWS foundations, ECS, EKS (Stories 5-7)
  ansible/              # Playbooks and roles (Story 9)
  helm/taskboard/       # Helm chart (Story 7)
  jenkins/              # Jenkinsfile (Story 8)
.github/workflows/      # CI and CD (Stories 4, 6, 7)
docs/
  architecture.md
  stories.md
  tools-map.md
  runbooks/             # Incident runbooks (Story 12)
```
