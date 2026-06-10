# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Safe Execution Mode (read first)

You are operating in safe execution mode. Before executing any command:

- Briefly explain what you're about to do in 1-2 simple sentences
- Use plain language, avoid jargon
- Say WHY, not just WHAT
- Then proceed with the action

This matters because this repo drives live AWS infrastructure (EKS, ECR, Terraform) where silent commands have real consequences. Always prefer clear reasoning before action. (This section is intentional and is referenced in `docs/claude-setup.md` — keep it.)

## What this repository is

This is a **teaching repository** — a DevOps + AIOps tutorial series, not a single deployable product. It is structured as documentation (`docs/`) plus the working artifacts those docs reference (`projects/`, `gitops/`). It also contains **intentional bugs and troubleshooting tasks** for learners (see `projects/Issues.md` and the "Bonus Challenge" in `README.md`). Do not "fix" things that look broken without checking whether the breakage is intentional.

The three top-level areas:

- `projects/boutique-microservices/` — the application: a React frontend + 6 Node.js backend services + PostgreSQL, runnable locally via Docker Compose.
- `projects/Infrastructure/` — Terraform that provisions the AWS target (VPC, EKS, ECR, ArgoCD, kube-prometheus-stack).
- `gitops/` — Kubernetes manifests (Kustomize) that ArgoCD syncs to the cluster.
- `projects/aiops-assistant/` — "Kira", a separate Python/Streamlit + AWS Bedrock Agent app for incident diagnosis (independent of the boutique app).

## Architecture: the boutique application

Request flow: **Frontend (3000) → Gateway (3001) → backend services → PostgreSQL (5432)**. The gateway is the only entry point clients talk to; it routes to the backend services by their `*_SERVICE_URL` env vars.

| Service | Port | DB |
|---|---|---|
| frontend (React/nginx) | 3000 | — |
| gateway | 3001 | — |
| auth | 3002 | auth_db |
| product-service | 3003 | products_db |
| order-service | 3004 | orders_db |
| orders | 3005 | orders_db |
| user-service | 3006 | users_db |

Each backend service owns its **own database** on a shared PostgreSQL instance (database-per-service). Connection is via a `DATABASE_URL` env var.

### Mixed TypeScript / JavaScript — important

Services are **not uniform**. Some are TypeScript, some plain JavaScript, and this changes how you build and run them:

- **TypeScript services** (`auth`, `gateway`, `product-service`, `user-service`, `orders`): entry is `src/index.ts`, dev runs via `nodemon src/index.ts`, and they have a `build` script (`tsc` → `dist/`). Their Dockerfiles do a multi-stage `npm run build`.
- **JavaScript services** (e.g. `order-service`): entry is `src/server.js`, no `tsc` build step, no `tsconfig.json`. Run directly with `node`/`nodemon`.
- You will also see stray `server.js` / `server-fixed.js` / `mock-*.js` files alongside the TS sources in some service `src/` dirs. Check `package.json` `main`/`scripts` to know which file actually runs before editing.

### Observability hooks

Each backend service exposes a `/metrics` endpoint via `prom-client`. Locally, Prometheus scrapes per `prometheus/prometheus.yml`. On EKS, a `ServiceMonitor` (`gitops/k8s/backend/service-monitor.yml`, labeled `release: kube-prometheus-stack`) tells the Prometheus Operator where to scrape, and the Grafana dashboard is auto-loaded from a ConfigMap labeled `grafana_dashboard: "1"`.

## Common commands

All boutique commands run from `projects/boutique-microservices/`. It uses **npm workspaces** (`frontend`, `backend/*`, `backend/services/*`).

```bash
# Local stack (fastest path) — builds images, starts all services + Postgres + Prometheus + Grafana
docker-compose -f docker-compose.yml up -d
docker-compose -f docker-compose.yml down

# Run without Docker
npm run install:all      # installs root, frontend, backend, and every service
npm run dev              # frontend + all backend services concurrently
npm run dev:backend      # backend services only
npm run dev:frontend     # React frontend only

# Build / test (delegate to frontend + backend workspaces)
npm run build
npm test
```

Working on a **single backend service** — cd into it (e.g. `backend/services/auth`):

```bash
npm run dev      # nodemon (TS: src/index.ts, JS: src/server.js)
npm run build    # tsc → dist/  (TypeScript services only)
npm start        # run the built/entry file
```

Note: most service `test` scripts are stubs (`echo "Error: no test specified" && exit 1`). There is no real backend test suite — the frontend uses `react-scripts test` (Jest).

### Infrastructure (Terraform)

```bash
cd projects/Infrastructure
terraform init
terraform plan
terraform apply --auto-approve      # ~15 min: VPC, EKS (eks-cluster, us-east-1), ECR x7, ArgoCD, monitoring
terraform destroy --auto-approve    # cleanup
```

### Kubernetes / GitOps

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
kubectl apply -k gitops/                              # apply all manifests via Kustomize
kubectl apply -f gitops/k8s/database/restore-job.yml  # seed the DB *after* the postgres pod is Ready
kubectl apply -f gitops/argo-cd.yml -n argocd         # register the ArgoCD Application
```

`gitops/kustomization.yml` is the single entry point listing every manifest; add new resources there. ArgoCD sync is currently **manual** (`syncPolicy: {}` in `gitops/argo-cd.yml`).

### CI/CD

`.github/workflows/ci.yml` is **manual only** (`workflow_dispatch`), not push-triggered. It builds all 7 service images in a matrix → pushes to ECR tagged with the commit SHA → a second job `sed`-rewrites image tags in `gitops/k8s/**` and commits back to `main`, which ArgoCD then syncs. Build context per service is `backend/services/<service>`; the frontend builds from `frontend/`.

### AIOps assistant (Kira) — separate Python app

```bash
cd projects/aiops-assistant
./setup-iam.sh                       # create IAM roles
./deploy.sh                          # create the Bedrock Agent + action groups
pip install -r requirements.txt
streamlit run app.py                 # UI on http://localhost:8501
```

Requires `BEDROCK_AGENT_ID` / `BEDROCK_AGENT_ALIAS_ID` in `.env`. See `projects/aiops-assistant/README.md` for the full Bedrock/Lambda setup and a detailed troubleshooting list.
