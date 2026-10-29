# Stories — work through these in order

One story = one branch = one pull request into `main`.
A story is done when every item under **Acceptance criteria** is true and
demonstrable, not just written. Paste evidence (command output, screenshot,
pipeline link, dashboard link) into the PR description.

---

## Story 1 — Repo foundations & branching

**Tools:** Git, GitHub
**Concepts you'll learn:** branching strategy, branch protection, PR workflow, `.gitignore` discipline.
**Depends on:** nothing.

### Tasks
1. Add `.gitignore` for Python, Terraform, Docker, and local secrets (`.env`, `*.tfstate`, `.terraform/`, `__pycache__/`, `.venv/`).
2. Add a PR template (`.github/pull_request_template.md`): what changed, how tested, evidence, rollback plan.
3. Enable branch protection on `main`: require a PR, require CI to pass (once Story 4 exists, come back and add it), no direct pushes.
4. Adopt the convention: `story/<n>-<short-name>`, e.g. `story/02-fastapi-app`.

### Acceptance criteria
- [ ] A PR merged to `main` using the template, from a feature branch — not a direct push.
- [ ] `.env` and Terraform state files are provably ignored (try to `git add` a dummy `.env` and show it is ignored).
- [ ] Branch protection is on for `main`.

### Stretch
- Add CODEOWNERS (just you, for now) and require one review — even if that review is your own second pass.

---

## Story 2 — Build TaskBoard API locally

**Tools:** Python, FastAPI, pytest, PostgreSQL, Docker Compose
**Concepts you'll learn:** API design, dependency injection for the DB, testing with a real DB, migrations, env-var config.
**Depends on:** Story 1.

### Tasks
1. Scaffold a FastAPI app: `GET /health` (checks DB), CRUD for tasks under `/tasks`.
2. Run PostgreSQL locally with Docker Compose. App runs on your machine, DB in a container.
3. Pick a migration mechanism (Alembic or versioned SQL) and record the choice in `docs/architecture.md`.
4. Write pytest tests: health check, create task, list tasks, update status, delete, 404 on missing task. Run tests against the Compose DB.
5. Structured JSON logging to stdout, with a request id per request. No secrets in logs.

### Acceptance criteria
- [ ] `curl localhost:8000/health` returns 200 with DB status.
- [ ] Full CRUD works via `curl` against the Compose Postgres.
- [ ] `pytest` passes, and tests do not depend on a hardcoded local database URL.
- [ ] App reads `DATABASE_URL` (and port) only from the environment.

### Stretch
- Add pagination to `GET /tasks` and a test for it.

---

## Story 3 — Dockerize it

**Tools:** Docker, Docker Compose
**Concepts you'll learn:** multi-stage builds, layer caching, image size, `.dockerignore`, non-root users, healthchecks.
**Depends on:** Story 2.

### Tasks
1. Write a multi-stage Dockerfile: build stage installs deps, runtime stage is slim, runs as a non-root user.
2. Add `.dockerignore` (`.git`, `.venv`, `__pycache__`, tests if you choose not to ship them).
3. Update Compose to run app + DB from the image, with a healthcheck on `/health`.
4. Compare image size before/after slimming and record it in the PR.

### Acceptance criteria
- [ ] `docker compose up` serves the API and passes `/health`, with no code mounted into the container.
- [ ] The image runs as non-root (`docker exec` + `whoami` proves it).
- [ ] Rebuilding after a code-only change reuses the dependency layer (show the cached step).

### Stretch
- Distroless or `python:slim` comparison, with the tradeoff written up.

---

## Story 4 — CI with GitHub Actions

**Tools:** GitHub Actions, Trivy, Python lint/test tooling
**Concepts you'll learn:** CI pipelines, caching, matrix, artifact handling, failing the build on scan findings, OIDC-ready secrets hygiene.
**Depends on:** Story 3.

### Tasks
1. Workflow on every PR: checkout, Python setup, dependency cache, lint (ruff or flake8), pytest against a Postgres service container, Docker build.
2. Add Trivy image scanning; fail the build on HIGH/CRITICAL findings (start with a documented exception list if a base-image finding blocks you — never silently ignore).
3. Push the image to a registry only on merge to `main`, tagged with the commit SHA (and `latest` only if you deliberately choose it — record why).

### Acceptance criteria
- [ ] A PR shows a green CI run: lint + tests + build + scan, all in one workflow.
- [ ] A deliberately introduced failing test fails the PR (prove it, then revert).
- [ ] No AWS or registry credentials are hardcoded in the workflow file.
- [ ] Branch protection (Story 1) now requires this workflow to pass.

### Stretch
- Add a Codecov-style coverage report, or a second job that runs the tests in parallel.

---

## Story 5 — Terraform AWS foundations

**Tools:** Terraform, AWS, Checkov
**Concepts you'll learn:** VPC design, IAM least privilege, remote state, variables per environment, IaC scanning, outputs.
**Depends on:** Story 4.

### Tasks
1. Terraform remote state: S3 bucket + DynamoDB lock table, created once (bootstrap) and never managed by the same state it stores.
2. Module/stack for: VPC (public + private subnets, NAT), ECR repository, RDS PostgreSQL (private subnets only, automated backups on), Secrets Manager secret for DB credentials, IAM roles for CI deploy and app runtime.
3. Variables for environment and region; `terraform fmt`, `validate`, and `plan` in CI (plan on PR, apply only on merge — or manually, with the plan pasted in the PR, for your first run).
4. Run Checkov on the Terraform code and fix or explicitly document every HIGH finding.

### Acceptance criteria
- [ ] `terraform plan` is clean and reviewable, with no secrets in the output.
- [ ] RDS is not publicly accessible; its security group allows traffic only from the app security group.
- [ ] State lives in S3, locking works (prove a second concurrent apply is blocked, or document the lock test).
- [ ] `terraform destroy` plan is understood and recorded — you will need it in Story 12.

### Stretch
- Rebuild one small resource (e.g. an S3 bucket) in CloudFormation and write a short comparison: when CloudFormation is the better fit, when Terraform is.

---

## Story 6 — Deploy to ECS Fargate

**Tools:** ECS Fargate, ALB, CloudWatch, AWS Secrets Manager, GitHub Actions
**Concepts you'll learn:** CD pipelines, task definitions, service discovery via ALB, health-check-driven deployments, log-driven debugging, rollback.
**Depends on:** Story 5.

### Tasks
1. Terraform: ECS cluster, task definition, Fargate service, ALB with `/health` health check, CloudWatch log group, app task role with read-only access to its Secrets Manager secret.
2. CD workflow: on merge to `main`, deploy the SHA-tagged image built in Story 4, wait for the service to stabilize.
3. Verify with `curl` against the ALB: `/health` and a full task CRUD round trip.
4. Practice a rollback: deploy a bad image (or bad env var), watch the health check fail, roll back to the previous task definition, record the steps.

### Acceptance criteria
- [ ] A merge to `main` deploys to ECS without manual AWS console clicks.
- [ ] The public URL serves `/health` and CRUD, backed by RDS.
- [ ] Logs are visible in CloudWatch, with request ids, and no secrets in them.
- [ ] A failed deploy is caught by the health check and rolled back — with evidence in the PR.

### Stretch
- Blue/green via CodeDeploy, or a second environment (`dev` vs `prod-like`) from the same Terraform with different variables.

---

## Story 7 — Kubernetes on EKS + Helm

**Tools:** EKS, kubectl, Helm, Terraform
**Concepts you'll learn:** Deployments, Services, Ingress, ConfigMaps/Secrets, probes, resource limits, HPA, Helm values per environment.
**Depends on:** Story 6. Local alternative: kind or minikube, if you want the Kubernetes learning without the EKS bill that week — say so in the PR and still do the Helm work.

### Tasks
1. Terraform: EKS cluster (small node group, or Fargate profile — pick one and record why), node IAM roles, Ingress controller.
2. Helm chart for TaskBoard: Deployment (2 replicas), Service, Ingress, ConfigMap, Secret (from Secrets Manager via external-secrets, or Helm-managed secret for learning — record the tradeoff), liveness/readiness probes on `/health`, CPU/memory requests and limits, HPA on CPU.
3. CD workflow: deploy the chart with the SHA-tagged image to EKS.
4. Load-test lightly (a simple loop or `k6` script) and watch HPA scale, then scale back down.

### Acceptance criteria
- [ ] `kubectl get pods,svc,ingress` shows a healthy deployment, and the Ingress URL serves `/health` and CRUD.
- [ ] Killing one pod is self-healed; the Service keeps serving.
- [ ] HPA scales up under load and back down after.
- [ ] The same image runs on ECS (Story 6) and EKS — only config differs.

### Stretch
- Argo CD or Flux GitOps: the cluster syncs from a Git branch instead of a push-based CD job. Compare the two models in the PR.

---

## Story 8 — Jenkins pipeline (parallel track)

**Tools:** Jenkins
**Concepts you'll learn:** Jenkins controllers vs agents, Jenkinsfile, shared libraries (at least the idea), Docker/Kubernetes agents, Jenkins vs GitHub Actions tradeoffs.
**Depends on:** Story 4 (you already have a working pipeline to mirror). Do not break the Actions pipeline — this is a parallel track.

### Tasks
1. Run Jenkins locally (Docker) or on a small EC2 instance via Story 9's Ansible, your choice — record it.
2. Write a Jenkinsfile that mirrors Story 4: lint, test against Postgres, build, Trivy scan.
3. Run it on a Docker agent (and, if time allows, a Kubernetes agent on the Story 7 cluster).
4. Write the comparison: cost, maintenance, startup time, secrets handling, and when you would pick Jenkins over Actions in a real team.

### Acceptance criteria
- [ ] A Jenkins build runs the same checks as the Actions workflow and fails on a failing test.
- [ ] No credentials are in the Jenkinsfile; they live in Jenkins credentials.
- [ ] The comparison write-up is in the repo, not just in your head.

### Stretch
- A Jenkins shared library with one reusable step (e.g. the Trivy scan), used by the pipeline.

---

## Story 9 — Configuration with Ansible

**Tools:** Ansible
**Concepts you'll learn:** inventory, playbooks, roles, idempotency, handlers, vault for Ansible secrets.
**Depends on:** Story 5 (you have AWS to configure against).

### Tasks
1. Inventory for a bastion/monitoring host (one small EC2 instance, or your local machine as `localhost` for the first pass — record which).
2. Role that installs Docker, node_exporter, and hardens SSH (no password auth, no root login), and is safe to run twice.
3. Playbook that provisions that host end to end, run twice to prove idempotency (second run: changed=0, or only expected changes).
4. Ansible Vault for any secret the playbook needs — never plaintext in the repo.

### Acceptance criteria
- [ ] Running the playbook twice produces no unexpected changes on the second run.
- [ ] `node_exporter` metrics are reachable from the host, and SSH hardening is verifiable.
- [ ] No secret is committed in plaintext; Vault usage is documented.

### Stretch
- Use Ansible to install the Jenkins controller from Story 8, instead of manual setup.

---

## Story 10 — Observability

**Tools:** Prometheus, Grafana, CloudWatch, Datadog / New Relic (optional), Splunk (optional)
**Concepts you'll learn:** RED/USE metrics, dashboards that answer questions, log correlation by request id, alert fatigue, APM vs metrics.
**Depends on:** Story 7 (EKS) and Story 6 (ECS/CloudWatch).

### Tasks
1. Prometheus + Grafana on EKS (Helm kube-prometheus-stack, or manual install — record the choice). Scrape TaskBoard `/metrics` and node/pod metrics.
2. One Grafana dashboard: request rate, error rate, p95 latency, CPU/memory, DB connections. One dashboard per environment.
3. Structured logs from the app correlated by request id in CloudWatch (ECS) and in Grafana (EKS, via Loki or CloudWatch datasource).
4. Optional, pick one: a Datadog or New Relic APM trial on the app, and a short comparison with Prometheus. Optional, if you have access: forward one log stream to Splunk and note what Splunk adds over CloudWatch.

### Acceptance criteria
- [ ] The dashboard shows real traffic, a real error spike (induce one), and recovery.
- [ ] You can go from a dashboard panel to the logs for the same request id.
- [ ] Every dashboard panel answers a question ("is it up?", "is it slow?", "is it saturated?") — remove panels that don't.

### Stretch
- Distributed tracing with OpenTelemetry: one trace from Ingress to DB query, visible in Grafana Tempo or Jaeger.

---

## Story 11 — Secrets & security hardening

**Tools:** AWS Secrets Manager, HashiCorp Vault, Snyk, IAM, network policies
**Concepts you'll learn:** secret rotation, dynamic secrets, least-privilege IAM, image and dependency scanning, Kubernetes network policy.
**Depends on:** Story 10.

### Tasks
1. Rotate the RDS credential via Secrets Manager and prove the app picks up the new secret (restart/reload path documented — no manual DB password edits in the console as the "fix").
2. Run Vault in dev mode, issue a dynamic database credential, and use it from a test script. Write up when Vault is worth it over Secrets Manager, and when it is overkill.
3. Add Snyk (or Dependabot + Trivy, if Snyk access is a blocker — record it) to CI for dependencies.
4. Audit IAM: CI deploy role, app task role, node role — remove any `*` permissions you can, and document any you must keep and why.
5. Add a Kubernetes NetworkPolicy: app pods can reach RDS and DNS, and nothing else by default.

### Acceptance criteria
- [ ] A rotated secret works end to end, with the rotation steps in a runbook.
- [ ] CI fails on a known-vulnerable dependency introduced deliberately (then reverted).
- [ ] NetworkPolicy is applied and a blocked-egress test proves it (e.g. a pod cannot reach an arbitrary external host).

### Stretch
- OIDC from GitHub Actions to AWS, replacing any long-lived AWS keys in GitHub secrets entirely.

---

## Story 12 — SLOs, alerting & incident drill

**Tools:** Grafana alerts, PagerDuty / Opsgenie, runbooks, Terraform destroy
**Concepts you'll learn:** SLOs and error budgets, symptom-based alerting, on-call, incident command, blameless postmortems, teardown discipline.
**Depends on:** Story 11 — this is the finale, do it last.

### Tasks
1. Define two SLOs: availability (e.g. 99.9% of `/health` checks succeed over 30 days, in learning terms: over the drill window) and latency (e.g. p95 < 300ms). Record the error budget math.
2. Wire Grafana alerts to PagerDuty or Opsgenie (free tier/trial): alert on error rate and latency, not on CPU alone. Every alert links to a runbook in `docs/runbooks/`.
3. Write runbooks: app down, DB connection exhaustion, high latency, bad deploy.
4. Game day: break it on purpose (bad image, DB max connections, or a killed deployment), follow the runbook, restore service, and time it. Write a blameless postmortem: timeline, what worked, what the alert/runbook got wrong, one action item.
5. Teardown: `terraform destroy` for ECS/EKS/RDS, delete ECR images if you choose, verify in the AWS console that nothing billable remains. Keep the repo, the docs, and the pipeline — the learning artifact is the repo, not a running cluster.

### Acceptance criteria
- [ ] Two SLOs are written down with error budgets, and Grafana shows them.
- [ ] A real alert fires, routes to PagerDuty/Opsgenie, and links to a runbook that actually helped.
- [ ] A postmortem exists for the game day, with at least one concrete follow-up.
- [ ] AWS teardown is verified — no EKS, RDS, ALB, or NAT gateway left running unless you deliberately choose to keep one and note its cost.

### Stretch
- Restore RDS from an automated backup into a new instance and prove the data is there, then delete the restored instance.

---

## AI track — Stories 13–16

Do these after Story 10 (observability) and Story 11 (secrets), and **before** Story 12 teardown — or re-provision with Story 5/7 Terraform first if you have already torn the cluster down. Same rule as before: one story = one branch = one pull request, done only on acceptance criteria with evidence in the PR. No app code is written for you; these are your build items, in order.

---

## Story 13 — MLOps core: train, register, serve, monitor

**Tools:** MLflow, scikit-learn, FastAPI, Docker, EKS (or kind/minikube), Terraform, GitHub Actions, Prometheus, Grafana
**Concepts you'll learn:** experiment tracking, model registry, model serving, retraining pipelines, drift monitoring, canary/rollback for models.
**Depends on:** Story 7 (EKS) and Story 10 (observability).

### Tasks
1. Train a small model on TaskBoard-like data (e.g. task-priority prediction, or any small classifier — the model itself is not the point, the pipeline is). Record ML fundamentals used: train/validation split, metrics, overfitting check, leakage check.
2. Track experiments and register the model in MLflow (tracking server + registry, versioned).
3. Serve the registered model behind a FastAPI endpoint on the EKS cluster (or kind/minikube), Dockerised, with Terraform for any new infra.
4. CI/CD for retrain-and-deploy: a GitHub Actions workflow that retrains, registers a new version, and deploys it, without manual console clicks.
5. Prometheus/Grafana for the model service: request rate, error rate, p95 latency, plus a simple drift signal (e.g. input distribution or prediction distribution shift).
6. Practice rollback/canary: promote a new model version gradually, then roll back to the previous version, and record the steps.

### Acceptance criteria
- [ ] An MLflow run and a registered, versioned model exist, with metrics recorded.
- [ ] The model endpoint serves predictions on the cluster and passes a health check.
- [ ] A CI run retrains and deploys a new model version; a bad version is caught and rolled back, with evidence in the PR.
- [ ] The Grafana dashboard shows latency, error rate, and a drift signal for the model service.

### Stretch
- Run the same model through SageMaker training + endpoint + Model Registry, and write a short comparison: self-managed (MLflow/KServe) vs managed (SageMaker) — cost, control, and when to pick each.

---

## Story 14 — Data platform: Databricks medallion pipeline

**Tools:** Databricks, PySpark, Delta Lake, Unity Catalog, Airflow or Databricks Jobs, Terraform
**Concepts you'll learn:** medallion architecture (Bronze → Silver → Gold), data-quality checks, data contracts, pipeline orchestration, cost per pipeline.
**Depends on:** Story 5 (Terraform). This story feeds Story 13's retraining — do it before you need fresh training data at scale.

### Tasks
1. Build a PySpark/Databricks pipeline in medallion form: Bronze (raw ingest), Silver (cleaned/validated), Gold (model-ready aggregates).
2. Add data-quality checks and a written data contract for the Gold table (schema, freshness, null/duplicate rules).
3. Orchestrate with Airflow or Databricks Jobs, with retries and alerting on failure.
4. Track cost per pipeline run, and record one cost optimisation you applied.
5. Prove the Gold table can be used as training input for Story 13's model.

### Acceptance criteria
- [ ] The pipeline runs end to end: Bronze → Silver → Gold, with evidence of each layer.
- [ ] A deliberately bad input fails a quality check and blocks the Gold update (prove it, then revert).
- [ ] The data contract is written down and enforced, not just described.
- [ ] Cost per pipeline run is recorded, with at least one optimisation noted.

### Stretch
- Unity Catalog governance: cluster policies, workspace permissions, and lineage for the Gold table, with the tradeoffs written up.

---

## Story 15 — RAG / LLMOps: retrieval, evaluation, guardrails

**Tools:** Amazon Bedrock (or a local open model), pgvector or Qdrant, LangChain or LlamaIndex, embeddings, reranking, evaluation harness, tracing
**Concepts you'll learn:** embeddings and chunking, hybrid search and reranking, prompt/model versioning, LLM evaluation, tracing, guardrails and PII handling.
**Depends on:** Story 13.

### Tasks
1. Embed a small document set (TaskBoard docs are a good start) and store embeddings in pgvector or Qdrant.
2. Build retrieval + reranking + generation: query → retrieve → rerank → generate, on Bedrock (or a local open model — record the choice and cost).
3. Add an evaluation harness: a handful of questions with expected answers, scored for relevance/faithfulness, latency, and cost.
4. Version prompts and models, and add tracing so you can follow one request from query to answer.
5. Add guardrails and PII redaction, and prove a PII test input is blocked/redacted.

### Acceptance criteria
- [ ] The RAG service answers questions from the document set, with retrieved sources visible in the response or logs.
- [ ] The evaluation harness runs and produces scores, not just vibes — a bad retrieval change moves the score, with evidence.
- [ ] Prompt and model versions are recorded, and a traced request is viewable end to end.
- [ ] A PII test input is blocked or redacted, with evidence in the PR.

### Stretch
- Add an agent step (one tool call, e.g. look up a task by ID) with LangGraph/LlamaIndex, and write up where agents help vs where plain RAG is enough.

---

## Story 16 — AI SRE hardening: SLOs, failover, AI cost

**Tools:** Grafana alerts, PagerDuty / Opsgenie, Karpenter / GPU scheduling concepts, vLLM / TGI concepts, FinOps reporting
**Concepts you'll learn:** SLOs for AI services, LLM failure modes, failover between models/providers, blue/green model promotion, cost per request/token, AI incident response.
**Depends on:** Story 15, and the SLO concepts from Story 12 (do this story before Story 12 teardown).

### Tasks
1. Define SLOs for the Story 15 RAG service: p95 latency, availability, error rate, and cost per request, with error-budget math.
2. Wire Grafana alerts to PagerDuty/Opsgenie, linked to a runbook in `docs/runbooks/`: RAG service down, latency spike, provider failure, cost spike.
3. Prove failover between two models/providers: kill or degrade the primary, watch traffic move, restore, and time it.
4. Practice blue/green model promotion: a new model version serves a fraction of traffic, then all traffic, with rollback documented.
5. Produce a FinOps-style AI cost report: cost per request/token, and one optimisation (caching, smaller model, batching) with measured effect.
6. Game day for the AI service: break it on purpose, follow the runbook, restore, and write a blameless postmortem with one concrete follow-up.

### Acceptance criteria
- [ ] SLOs for the RAG service are written down with error budgets and shown in Grafana.
- [ ] A real alert fires, routes to PagerDuty/Opsgenie, and links to a runbook that actually helped.
- [ ] Failover between models/providers is proved, with timings in the PR.
- [ ] Cost per request/token is reported, with at least one measured optimisation.
- [ ] A postmortem exists for the AI game day, with a concrete follow-up.
- [ ] Story 12 teardown is still the final step after this story — do not tear down before these AI stories are done, or re-provision first.
