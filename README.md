# End-to-End GitOps CI/CD Pipeline

> A production-grade CI/CD pipeline where a single `git push` is the only manual step.
> Code goes from laptop to live Kubernetes deployment automatically — no manual `kubectl apply`, ever.

---

## Live Pipeline Status

![CI Pipeline](https://github.com/irfanjat/myapp/actions/workflows/ci.yml/badge.svg)

---

## What This Project Does

This project implements a complete **GitOps workflow** using two repositories, GitHub Actions for CI, and ArgoCD for CD — the same pattern used by engineering teams at scale.

```
Developer pushes code
        │
        ▼
┌─────────────────────┐
│   GitHub Actions CI  │
│  ┌───────────────┐  │
│  │  1. Run tests  │  │
│  │  2. Build image│  │
│  │  3. Push GHCR  │  │
│  │  4. Update cfg │  │
│  └───────────────┘  │
└─────────────────────┘
        │
        ▼ commits new SHA tag to myapp-config
┌─────────────────────┐
│      ArgoCD          │
│  Watches config repo │
│  Detects change      │
│  Syncs cluster       │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Kubernetes Cluster  │
│  Rolling deployment  │
│  Zero downtime       │
└─────────────────────┘
```

---

## Architecture

### Two-Repo Pattern (GitOps Best Practice)

| Repo | Purpose |
|------|---------|
| `myapp` | Application source code, Dockerfile, tests, CI pipeline |
| `myapp-config` | Kubernetes manifests — the single source of truth for cluster state |

Keeping them separate means:
- You can update replicas or resource limits without triggering a full rebuild
- The config repo is a clean audit log of every deployment — who changed what and when
- ArgoCD watches the config repo, not the app repo — clean separation of concerns

### CI Pipeline — 3 Jobs in Sequence

```
test (11s) ──► build-and-push (1m 21s) ──► update-config (5s)
```

- **test** — runs `pytest` against the Flask app. Pipeline stops here if any test fails
- **build-and-push** — builds Docker image, tags it with the exact commit SHA, pushes to GHCR
- **update-config** — checks out `myapp-config`, `sed`-replaces the image tag, commits and pushes

### CD — ArgoCD

ArgoCD runs inside the cluster and polls `myapp-config` every 3 minutes. The moment it detects a new commit (a new image SHA), it applies the updated manifests and performs a rolling deployment — old pods terminate only after new pods pass readiness probes.

---

## Tech Stack

| Tool | Role |
|------|------|
| **Python / Flask** | Application — 2 endpoints (`/` and `/health`) |
| **pytest** | Unit tests — 3 tests, run on every push and PR |
| **Docker** | Containerisation — `python:3.12-slim` base image |
| **GitHub Actions** | CI orchestrator — test, build, push, update config |
| **GHCR** | Container registry — images tagged with commit SHA |
| **ArgoCD v3.3.4** | GitOps CD controller — watches Git, syncs cluster |
| **Kubernetes (kind)** | Container orchestration — 2 replicas, rolling updates |
| **Helm** | Used to install ArgoCD on the cluster |

---

## Key Engineering Decisions

### Immutable image tags
Every Docker image is tagged with the full Git commit SHA — never `latest` in production.
This means every running deployment is traceable to an exact line of code.

```yaml
# deployment.yaml — updated automatically by CI
image: ghcr.io/irfanjat/myapp:6f5b0192660c042861cc0f97cfa0c6504b94199e
```

### ArgoCD self-heal
```yaml
syncPolicy:
  automated:
    prune: true       # removes resources deleted from Git
    selfHeal: true    # reverts any manual kubectl changes automatically
```
If someone manually runs `kubectl scale deployment myapp --replicas=5`,
ArgoCD will detect the drift and revert to the Git-defined value within minutes.

### Health-gated rolling updates
Pods only receive traffic after passing the `/health` readiness probe.
Kubernetes will not terminate old pods until new pods are confirmed healthy.

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Branch protection
`main` requires a passing CI run before any merge. No broken code reaches the cluster.

---

## Repository Structure

```
myapp/                          # This repo — application source
├── .github/
│   └── workflows/
│       └── ci.yml              # Full CI pipeline definition
├── app.py                      # Flask application
├── test_app.py                 # pytest test suite
├── requirements.txt            # Python dependencies
├── Dockerfile                  # Container build instructions
└── .dockerignore

myapp-config/                   # Separate repo — Kubernetes manifests
└── k8s/
    ├── namespace.yaml           # myapp namespace
    ├── deployment.yaml          # 2 replicas, resource limits, health probes
    └── service.yaml             # ClusterIP service on port 80
```

---

## How to Run Locally

```bash
# Clone the repo
git clone https://github.com/irfanjat/myapp.git
cd myapp

# Install dependencies
pip install -r requirements.txt

# Run tests
pytest test_app.py -v

# Run the app
python app.py

# Or with Docker
docker build -t myapp:local .
docker run -p 5000:5000 myapp:local
```

Hit `http://localhost:5000` — you should see:

```json
{
  "message": "Hello from myapp!",
  "status": "running",
  "version": "1.0.0"
}
```

---

## Deploying to Your Own Cluster

**Prerequisites:** `kubectl`, `helm`, `argocd` CLI, a running Kubernetes cluster

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 3. Apply the ArgoCD application manifest
kubectl apply -f https://github.com/irfanjat/myapp-config/blob/main/argocd-app.yaml

# 4. Watch the sync
argocd app get myapp
kubectl get pods -n myapp -w
```

---

## Results

| Metric | Before | After |
|--------|--------|-------|
| Manual deployment steps | ~8 commands | 0 — just `git push` |
| Time from push to live | ~10 minutes manual | ~2 minutes automated |
| Deployment traceability | "I think it was that commit" | Exact SHA in every manifest |
| Drift detection | None | ArgoCD alerts + auto-heals within minutes |

---

## What I Would Add Next

- **Helm chart** — replace raw manifests with a parameterised Helm chart
- **Staging environment** — separate namespace with auto-deploy, production requires manual approval
- **Slack notifications** — ArgoCD fires a message on every sync success or failure
- **Image vulnerability scanning** — Trivy scans the image in CI before it gets pushed
- **Horizontal Pod Autoscaler** — scale replicas automatically based on CPU load

---

## Author

**Irfan Jat** — CS student building production-grade DevOps infrastructure.

[![GitHub](https://img.shields.io/badge/GitHub-irfanjat-181717?logo=github)](https://github.com/irfanjat)
