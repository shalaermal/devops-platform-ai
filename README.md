# DevOps Platform

A production-grade DevOps platform built on Kubernetes, simulating real-world company infrastructure. This project covers the full lifecycle of modern cloud infrastructure — from provisioning to deployment, monitoring, automated operations, and **AI-powered incident response**.

---

## ✨ Highlights

- **Full GitOps pipeline** — every change goes through Git, ArgoCD syncs automatically
- **AI Incident Response** — when alerts fire, an AI agent analyzes logs and suggests fixes in Slack
- **Production-grade observability** — Prometheus, Grafana, Loki, AlertManager all integrated
- **Secure by design** — OIDC authentication, RBAC, Network Policies, no static credentials
- **Fully automated CI/CD** — security scans, Docker builds, ECR push, auto-deploy on merge

---

## Architecture Overview

```
GitHub (Source of Truth)
        |
        |-- Push to main
        |
GitHub Actions (CI/CD)
        |
        |-- Build & Push Docker Images
        |-- ECR (AWS Container Registry)
        |
ArgoCD (GitOps)
        |
        |-- Sync to Kubernetes Cluster
        |
Kind Cluster (Local Kubernetes)
        |
        |-- ingress-nginx (Traffic Management)
        |-- podinfo (Demo Workload)
        |-- Prometheus (Metrics Collection)
        |-- Grafana (Metrics Visualization)
        |-- Loki (Log Aggregation)
        |-- AlertManager (Alerting)
        |-- AI Agent (Incident Response) 
        |-- HPA (Auto Scaling)
        |-- CronJobs (Scheduled Tasks)
        |
AWS (Cloud Infrastructure)
        |-- S3 (Storage & Artifacts)
        |-- ECR (Container Registry)
        |-- IAM + OIDC (Secure Authentication)
```

---

## Stack

| Category | Technology |
|---|---|
| Cloud | AWS (S3, ECR, IAM) |
| Infrastructure as Code | Terraform |
| Container Orchestration | Kubernetes (kind) |
| CI/CD | GitHub Actions |
| GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana |
| Logging | Loki + Promtail |
| Alerting | AlertManager + Slack |
| **AI Incident Response** | **Groq AI (Llama 3)** |
| Load Testing | k6 |
| Traffic Management | ingress-nginx + MetalLB |
| Auto Scaling | HPA + metrics-server |
| Scheduled Jobs | Kubernetes CronJobs |
| Containerization | Docker |

---

## Project Structure

```
devops-platform/
├── .github/
│   └── workflows/
│       ├── ci.yaml              # CI Pipeline (runs on PR)
│       └── cd.yaml              # CD Pipeline (runs on merge to main)
├── apps/
│   ├── frontend/                # React application
│   ├── api/                     # Python FastAPI
│   ├── worker/                  # Python background worker
│   └── ai-agent/                # AI Incident Response Agent (Python Flask)
├── kubernetes/
│   ├── apps/
│   │   ├── podinfo/             # Demo workload manifests
│   │   ├── ai-agent/            # AI Agent deployment + service
│   │   └── rbac/                # RBAC ArgoCD application
│   ├── argocd/                  # ArgoCD network policies
│   ├── cronjobs/                # Scheduled job definitions
│   ├── ingress-nginx/           # Ingress controller values
│   ├── loki/                    # Logging stack values
│   ├── monitoring/              # Prometheus + Grafana + AlertManager values
│   ├── promtail/                # Promtail values
│   └── rbac/                    # RBAC roles and bindings
├── k6/
│   └── load-test.js             # Load testing scripts
├── scripts/
│   └── start-platform.sh        # Local development helper
├── terraform/
│   ├── environments/
│   │   └── dev/                 # Dev environment configuration
│   └── modules/
│       ├── ecr/                 # ECR repositories module
│       ├── iam/                 # IAM roles and policies module
│       ├── kind/                # Kind cluster module
│       └── s3/                  # S3 bucket module
└── docs/
    └── runbooks/                # Operational runbooks
```

---

## Infrastructure

### AWS (Terraform)

All AWS resources are provisioned and managed with Terraform using a modular structure.

**S3** — stores Terraform state and build artifacts. Configured with versioning and AES256 server-side encryption.

**ECR** — private container registry for Docker images. Four repositories: `frontend`, `api`, `worker`, and `ai-agent`. Each has a lifecycle policy retaining only the 10 most recent images with automatic vulnerability scanning on push.

**IAM** — GitHub Actions authenticates with AWS using OIDC (OpenID Connect), eliminating long-lived access keys. The role follows least privilege, granting only ECR and S3 permissions.

### Kubernetes Cluster (kind)

Provisioned locally using kind (Kubernetes in Docker) and managed through Terraform. One control-plane node and two worker nodes with role-based assignment:

- `worker` → application workloads
- `worker2` → monitoring stack (Prometheus, Grafana, Loki, AlertManager)

---

## CI/CD Pipeline

### CI Pipeline (Pull Requests)

Triggered on every PR targeting `main`. Runs Trivy security scans on filesystem and Docker images for all services.

### CD Pipeline (Merge to main)

Triggered on merge to `main`. Builds and pushes all four Docker images to ECR tagged with the Git commit SHA. ArgoCD auto-deploys on detecting the change.

```
PR opened
    |
CI Pipeline (Trivy security scan + Docker build)
    |
Code review and approval
    |
Merge to main
    |
CD Pipeline (build + push to ECR)
    |
ArgoCD detects Git change
    |
Automatic deployment to cluster
```

---

## Monitoring

### Prometheus

Collects metrics from all cluster components. ServiceMonitors automatically discover and scrape application metrics. Configured with 7-day retention.

### Grafana

Custom dashboards built using PromQL and LogQL:

- **podinfo CPU Usage %** — real-time CPU per pod
- **podinfo RAM Usage (MB)** — memory consumption per pod
- **podinfo Pod Restarts** — restart tracking
- **podinfo Logs** — live log streaming from Loki
- **RED Dashboard** — Rate, Errors, Duration metrics

### AlertManager

Alert rules configured:

| Alert | Condition |
|---|---|
| `PodCrashLooping` | restart rate > 0 for 1 minute |
| `PodNotReady` | not ready for 2+ minutes |
| `HighCPUUsage` | CPU > 80% for 2 minutes |
| `HighMemoryUsage` | memory > 85% for 2 minutes |

Alerts routed to Slack with resolved notifications.

### Loki + Promtail

Loki aggregates logs cluster-wide. Promtail runs as a DaemonSet — one collector per node.

---

## 🤖 AI-Powered Incident Response

The most unique feature of this platform. When AlertManager fires an alert, the AI Agent automatically:

1. Receives the webhook from AlertManager
2. Fetches recent pod logs from Loki
3. Sends alert context + logs to **Groq AI (Llama 3)**
4. Posts a structured analysis to Slack

```
Alert fires
    |
AlertManager ──────────────────────────────► Slack
    |                                         (standard notification)
    ▼
AI Agent webhook
    |
Fetch logs from Loki
    |
Groq AI (Llama 3.1) analysis
    |
    ▼
Slack message:
┌─────────────────────────────────────────┐
│ 🤖 AI Incident Analysis: PodCrashLooping│
│ Namespace: argocd | Pod: argocd-server  │
│                                         │
│ Root Cause:                             │
│ Connectivity timeout to Kubernetes API  │
│ server causing crash loop.              │
│                                         │
│ Immediate Fix:                          │
│ kubectl delete pod argocd-server-xxx    │
│                                         │
│ Long-term Recommendation:               │
│ Implement pod anti-affinity rules.      │
└─────────────────────────────────────────┘
```

The AI Agent is a containerized Python Flask service deployed in Kubernetes. API keys are stored as Kubernetes Secrets — never in Git.

**Tech stack:** Python + Flask + Groq API + Loki API + Slack Webhooks

---

## RBAC

| Role | Scope | Permissions |
|---|---|---|
| `developer` | podinfo namespace | get, list, watch pods/logs/deployments |
| `ops` | podinfo namespace | full access to all resources |
| `readonly` | cluster-wide | get, list, watch all resources |

---

## Network Policies

Network policies defined for `podinfo`, `monitoring`, and `argocd` namespaces to restrict inter-namespace traffic.

> **Note:** Enforcement requires Calico or Cilium CNI. The default kind CNI (kindnet) does not enforce Network Policies. In production (EKS, GKE), these policies are enforced automatically.

---

## GitOps with ArgoCD

ArgoCD continuously monitors the Git repository and reconciles cluster state. Manual changes to the cluster are automatically reverted.

Applications managed:

| Application | Path |
|---|---|
| `podinfo` | `kubernetes/apps/podinfo` |
| `argocd-config` | `kubernetes/argocd` |
| `rbac` | `kubernetes/rbac` |

---

## Auto Scaling

HPA configured for podinfo with CPU target 50%, min 2 replicas, max 4. Validated with k6 load testing — full scale-up/down cycle visible in Grafana.

---

## Local Development

### Prerequisites

- WSL2 with Ubuntu 24.04
- Docker Desktop with WSL2 integration enabled
- kubectl, kind, terraform, helm, aws cli, git, k6, ngrok

### Quick Start

```bash
# Provision infrastructure
terraform -chdir=terraform/environments/dev apply

# Start monitoring stack
bash scripts/start-platform.sh

# External access
ngrok http 3000
```

### Access Services

```bash
# Grafana
kubectl port-forward -n monitoring deploy/prometheus-grafana 3000:3000

# Prometheus
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9091:9090 -n monitoring

# ArgoCD
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

### Run Load Test

```bash
kubectl port-forward svc/podinfo 9898:9898 -n podinfo &
k6 run k6/load-test.js
```

---

## Quick Recovery (Monitoring Stack)

```bash
# 1. Prometheus + Grafana
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values kubernetes/monitoring/values.yaml

# 2. Loki
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --values kubernetes/loki/values.yaml

# 3. Promtail
helm upgrade --install promtail grafana/promtail \
  --namespace monitoring \
  --values kubernetes/promtail/values.yaml

# 4. Loki datasource in Grafana
kubectl exec -n monitoring deploy/prometheus-grafana -c grafana -- \
  curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Loki","type":"loki","url":"http://loki-gateway.monitoring.svc.cluster.local","access":"proxy","isDefault":false}' \
  "http://admin:devops123@localhost:3000/api/datasources"

# 5. AI Agent secrets
kubectl create secret generic groq-api-key \
  --from-literal=api-key=YOUR_GROQ_API_KEY \
  --namespace=default

kubectl create secret generic slack-webhook \
  --from-literal=url=YOUR_SLACK_WEBHOOK_URL \
  --namespace=default
```

---

## Key Concepts Demonstrated

- Infrastructure as Code with Terraform (modular, multi-environment)
- Kubernetes cluster management — deployments, services, ingress, autoscaling, scheduled jobs
- GitOps with ArgoCD — Git as single source of truth
- Full observability — metrics, logs, alerts, dashboards
- **AI-powered incident response** — automated root cause analysis and fix suggestions
- Secure cloud authentication via OIDC — no static credentials
- Automated CI/CD with security scanning (Trivy)
- RBAC — role-based access control
- Network policies for namespace isolation

---

## Security Notice

Before deploying, replace all placeholder values:

- `kubernetes/monitoring/values.yaml` — replace `adminPassword` with a strong password
- `kubernetes/monitoring/values.yaml` — replace `slack_api_url` with your Slack webhook URL
- Create Kubernetes secrets for `groq-api-key` and `slack-webhook` — never commit API keys to Git

---

