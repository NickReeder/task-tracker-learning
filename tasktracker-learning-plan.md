# TaskTracker Learning Plan: Docker, Helm & DevSecOps

A 12-week beginner plan (about 6-8 hours per week) built around deploying one sample app.

| Layer | Technology |
|---|---|
| Front end | Angular |
| Back end | FastAPI (Python) |
| Database | PostgreSQL (local container first, AWS RDS in the capstone) |
| CI/CD and DevSecOps | Azure DevOps (Repos + Pipelines) |
| Packaging and deploy | Docker, Kubernetes, Helm |
| Cloud | AWS (ECR, EKS, RDS, Secrets Manager) |

**How to use this file:** tick the boxes as you finish items, and add notes in the Progress Log at the bottom.

---

## The sample app: TaskTracker

```
Browser -> Angular (nginx container) -> FastAPI (container) -> PostgreSQL
                  all deployed to Kubernetes with Helm
                  built, scanned, and shipped by Azure DevOps
```

Repo layout:

```
tasktracker/
├─ frontend/            # Angular app + Dockerfile + nginx.conf
├─ backend/             # FastAPI app + Dockerfile + alembic/ (DB migrations)
├─ charts/tasktracker/  # Helm chart (Chart.yaml, values*.yaml, templates/)
└─ azure-pipelines.yml
```

API contract (kept stable so every lesson builds on the last):

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness/readiness checks (used by Kubernetes and smoke tests) |
| GET | `/tasks` | List tasks |
| POST | `/tasks` | Create a task (`{"title": "..."}`) |

**Two things to know upfront**

- Helm deploys to Kubernetes, so a short Kubernetes module (Week 4) comes before it.
- RDS normally sits in a private network your laptop cannot reach. Use a local Postgres container until the capstone, when the cluster runs inside AWS.

---

## Roadmap at a glance

| Weeks | Phase | Outcome |
|---|---|---|
| 1 | Foundations | Working app on your laptop, code in Azure Repos |
| 2-3 | Docker | Both services containerized, running with Compose |
| 4 | Kubernetes bridge | App running on a local cluster via raw YAML |
| 5-6 | Helm | One chart deploying the whole app with dev/prod configs |
| 7-8 | Azure DevOps CI/CD | Push code, build, push image to registry, deploy |
| 9-11 | DevSecOps deep dive | Security gates in the pipeline and cluster |
| 12 | Capstone | Full pipeline to EKS + RDS, then teardown |

Security is practiced in every phase (look for the lock icon), then consolidated and automated in Weeks 9-11.

---

## Tools checklist

- [ ] Python 3.12+ and `pip`
- [ ] Node.js (current LTS) and npm
- [ ] Angular CLI (`npm install -g @angular/cli`)
- [ ] Git
- [ ] VS Code (or another editor)
- [ ] Docker Desktop (needed from Week 2)
- [ ] `kubectl` and a local cluster: kind, minikube, or Docker Desktop Kubernetes (Week 4)
- [ ] Helm 3 (Week 5)
- [ ] Azure DevOps organization and project (free tier)
- [ ] AWS account with a **budget alert set** (needed by Week 7 for ECR)

---

## Phase 0: Foundations (Week 1)

**Learn:** shell basics, Git (branch, commit, pull request), how HTTP, ports and environment variables work, the 12-factor idea of "config lives in the environment".

**Lesson 1 parts**
- [ ] 1.1 Install and verify tools
- [ ] 1.2 Concepts: HTTP, ports, environment variables
- [ ] 1.3 Build the FastAPI backend (`/health`, `/tasks`, in-memory)
- [ ] 1.4 Build the Angular frontend and connect it via a dev proxy
- [ ] 1.5 Git, `.gitignore`, push to Azure Repos, complete one pull request

**Security checkpoint:** `.gitignore` covers `.env`, `.venv`, and `node_modules`. Never commit a password, even a fake one, because the habit matters more than the value.

**Done when:** the frontend talks to the backend on your machine, and the code lives in Azure Repos.

---

## Phase 1: Docker (Weeks 2-3)

### Week 2: Containers and the backend
**Learn:** images vs. containers, layers and caching, `docker build/run/logs/exec`, port mapping, environment variables, `.dockerignore`.

- [ ] Write a Dockerfile for FastAPI (slim base image, dependency layer cached, non-root user)
- [ ] Build, run, and inspect the container; break it on purpose and read the logs
- [ ] Explain layer caching by reordering `COPY` lines and timing rebuilds

**Security checkpoint:** why non-root matters, why `slim` images have a smaller attack surface, why base image versions get pinned.

### Week 3: Frontend, Compose, and databases
**Learn:** multi-stage builds, Docker networks and volumes, `docker compose`.

- [ ] Multi-stage Angular Dockerfile (Node build stage, nginx runtime stage)
- [ ] nginx config that serves the app and proxies `/api` to the backend
- [ ] `compose.yaml` with frontend, backend, and local Postgres
- [ ] Add SQLAlchemy and Alembic so tasks persist across restarts

**Security checkpoint:** pass DB credentials as environment variables, never bake them into images. Run `docker history` to see how secrets leak into layers.

**Done when:** `docker compose up` gives you the full app with persistent data.

---

## Phase 2: Kubernetes bridge (Week 4)

**Learn:** Pod, Deployment, Service, ConfigMap, Secret, Ingress, readiness/liveness probes, `kubectl get/describe/logs/apply`.

- [ ] Create a local cluster
- [ ] Deploy TaskTracker with hand-written YAML (about 6-8 files)
- [ ] Wire `/health` into readiness and liveness probes
- [ ] Notice the repetition. That is why Helm exists.

**Security checkpoint:** Kubernetes Secrets are base64-encoded, not encrypted. Add `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`, drop all capabilities) and resource limits.

**Done when:** the app runs in the cluster and you can explain what each YAML file does.

---

## Phase 3: Helm (Weeks 5-6)

### Week 5: Helm fundamentals
**Learn:** chart, release, repository, `helm install/upgrade/rollback/uninstall`, `values.yaml`, `--set`.

- [ ] Install a public chart to see how consumption works
- [ ] `helm create`, then convert your backend YAML into a chart
- [ ] Practice `helm template` and `helm lint`
- [ ] Break a deploy on purpose and run `helm rollback`

### Week 6: Real-world chart patterns
**Learn:** Go templating (`{{ .Values.x }}`), `_helpers.tpl`, conditionals and loops, `values-dev.yaml` vs. `values-prod.yaml`, hooks, subcharts/umbrella charts, `helm test`, packaging to an OCI registry.

- [ ] Add the frontend to the chart with per-environment replicas, resources, and image tags
- [ ] Run Alembic migrations as a pre-upgrade Helm hook Job
- [ ] Add a `helm test` that calls `/health`

**Security checkpoint:** never put real secrets in `values.yaml`. Reference an existing Kubernetes Secret instead. Add default `securityContext` values so every release inherits them.

**Done when:** `helm upgrade --install tasktracker ./charts/tasktracker -f values-dev.yaml` deploys the full app and you can roll it back.

---

## Phase 4: Azure DevOps CI/CD (Weeks 7-8)

Image registry: **Amazon ECR**, so images, cluster, and database all live in AWS.

### Week 7: Continuous Integration
**Learn:** YAML pipeline anatomy (trigger, stages, jobs, steps), agents, variables and variable groups, service connections, branch policies and PR validation.

- [ ] CI pipeline: backend tests, Angular build, Docker builds, push to ECR tagged `$(Build.BuildId)`
- [ ] PR-validation pipeline
- [ ] `helm lint` and `helm package` in the pipeline

**Tip:** new Azure DevOps organizations often must request free Microsoft-hosted parallelism. A self-hosted agent on your own machine works fine for learning.

### Week 8: Continuous Deployment
**Learn:** Environments, approvals and checks, deployment jobs, AWS service connection (AWS Toolkit for Azure DevOps).

- [ ] Auto-deploy to `dev` with `helm upgrade --install --atomic`
- [ ] `prod` stage with a manual approval gate
- [ ] Smoke test after each deploy

```yaml
stages:
- stage: Build_Test      # lint, unit tests
- stage: Package         # docker build, push to ECR, helm package
- stage: Deploy_Dev      # helm upgrade --install, smoke test
- stage: Deploy_Prod     # environment approval, then helm upgrade --install --atomic
```

**Security checkpoint:** secrets in secret variables or a Key Vault-linked variable group. Restrict who can edit pipelines and approve prod. Prefer short-lived credentials over long-lived access keys.

**Done when:** merging to `main` deploys to dev with no manual steps.

---

## Phase 5: DevSecOps deep dive (Weeks 9-11)

Core idea: **shift security left**. Catch problems in the pull request, not in production.

### Week 9: Scan code and dependencies
- [ ] Secret scanning with Gitleaks (pipeline and pre-commit hook)
- [ ] SAST: Bandit and Semgrep (Python), ESLint security rules (Angular)
- [ ] SCA: `pip-audit` and `npm audit`
- [ ] Map the OWASP Top 10 to TaskTracker (where could injection or broken access control appear?)

### Week 10: Scan containers and charts
- [ ] Trivy image scans that fail the build on high/critical CVEs
- [ ] Chart scanning (Checkov, kube-linter, or Trivy config) on `helm template` output
- [ ] SBOM with Syft, image signing with Cosign
- [ ] Practice triage: suppress with justification, never blindly ignore

### Week 11: Harden runtime and database
- [ ] Pod Security Standards, NetworkPolicies, Kyverno policies
- [ ] AWS Secrets Manager + External Secrets Operator (no secrets in Git or logs)
- [ ] RDS: private subnets, tight security groups, TLS, encryption at rest, least-privilege app DB user separate from the migration user
- [ ] OWASP ZAP baseline scan against dev as a pipeline step
- [ ] Azure DevOps hardening: branch policies, required reviewers, protected environments

**Done when:** a deliberately vulnerable change (hardcoded secret, outdated base image) is blocked by your pipeline.

---

## Phase 6: Capstone (Week 12)

- [ ] Provision a small EKS cluster and an RDS PostgreSQL instance in the same VPC
- [ ] Point Helm values at RDS and deploy through the full pipeline: PR checks, scans, build, sign, dev, approval, prod
- [ ] Write a one-page threat model for TaskTracker
- [ ] Write a README documenting the pipeline's security gates
- [ ] **Tear everything down** (EKS and RDS cost money while running)

---

## How to study

- **Type everything yourself.** Copy-paste teaches far less than typing and debugging.
- **Break things on purpose.** A failed probe, a bad image tag, or a wrong password teaches more than a clean run.
- **Keep a "what I'd explain to a friend" note per week.** If you can't explain it simply, revisit it.

## References

- Docker Get Started: https://docs.docker.com/get-started/
- Helm docs: https://helm.sh/docs/
- Azure Pipelines docs: https://learn.microsoft.com/en-us/azure/devops/pipelines/
- Kubernetes basics: https://kubernetes.io/docs/tutorials/kubernetes-basics/
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/

---

## Progress log

| Week | Started | Finished | What I'd explain to a friend | Where I got stuck |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |
| 6 | | | | |
| 7 | | | | |
| 8 | | | | |
| 9 | | | | |
| 10 | | | | |
| 11 | | | | |
| 12 | | | | |
