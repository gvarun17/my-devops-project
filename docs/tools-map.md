# Must-know tools map

From the must-know list. "Core" means you build with it; "touched" means you
use it enough to explain it, even if it is not the primary tool in that story.

| Tool | Stories | How you use it |
|---|---|---|
| Git, GitHub | 1, and every story | Branches, PRs, protection, reviews |
| Python, Bash | 2, and every story | App code, scripts, pipeline glue |
| FastAPI, pytest | 2 | App + tests |
| PostgreSQL | 2, 5 | Local Compose DB, then RDS |
| Docker, Docker Compose | 2, 3 | Local DB, multi-stage image |
| GitHub Actions | 4, 6, 7 | CI, then CD to ECS and EKS |
| Trivy | 4, 11 | Image scanning in CI |
| Terraform, AWS | 5, 6, 7 | VPC, IAM, ECR, RDS, ECS, EKS |
| CloudFormation | 5 (stretch) | Rebuild one small piece (e.g. an S3 bucket) in CloudFormation and compare |
| AWS Secrets Manager | 5, 6, 11 | DB credentials, app secrets |
| ECS Fargate, ALB | 6 | First cloud deployment |
| CloudWatch | 6, 10 | ECS logs and metrics |
| Kubernetes, kubectl | 7 | EKS workloads, probes, HPA |
| Helm | 7 | Chart, values per environment |
| Jenkins | 8 | Jenkinsfile mirroring the Actions pipeline |
| Ansible | 9 | Bastion/monitoring host config, roles |
| Prometheus, Grafana | 10 | Metrics, dashboards, alerts |
| Datadog / New Relic | 10 (optional) | One APM trial, compare with Prometheus |
| Splunk | 10 (optional) | Forward a log stream if you have access; otherwise note it as read-only learning |
| HashiCorp Vault | 11 | Dev-mode Vault, dynamic secret, rotation |
| Snyk, Checkov | 5, 11 | IaC and dependency scanning |
| PagerDuty / Opsgenie | 12 | Alert routing, on-call schedule (free tier / trial) |
| SLOs, runbooks | 12 | Error budget, incident drill, teardown |

Not in the core path, deliberately: a frontend, a cache, a queue, and a
service mesh. Add them only after Story 12, as stretch work — they are
nice-to-have, not must-know, and they dilute the core if added early.
