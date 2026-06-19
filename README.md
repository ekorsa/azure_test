# Azure DevOps Demo — AKS + Helm + Prometheus + Grafana

Demo DevOps project: provisioning a Kubernetes cluster on Azure (AKS) via Terraform and deploying an application with a built-in monitoring stack via Helm.

> Демонстрационный DevOps-проект: поднятие Kubernetes-кластера в Azure (AKS) через Terraform и деплой приложения с встроенным стеком мониторинга через Helm.

---

## Stack / Стек

| Layer | Technologies |
|---|---|
| Infrastructure | Terraform, Azure AKS |
| Deploy | Helm 3 |
| Application | FastAPI (ghcr.io/ekorsa/fastapi_docker_test) |
| Monitoring | Prometheus v2.45, Grafana 10.2 (sidecar) |
| Ingress | nginx-ingress |
| Backup | Kubernetes CronJob (Alpine) |
| Local development | Minikube |

---

## Architecture / Архитектура

```
                  ┌─────────────────────────────────────────┐
                  │          Kubernetes Pod                  │
                  │                                         │
  Browser ──────► │  [Nginx Ingress]                        │
                  │       │                                 │
                  │       ├──► FastAPI  :8000               │
                  │       │       │ metrics                 │
                  │       │       ▼                         │
                  │       ├──► Prometheus :9090  ──► PVC    │
                  │       │       │                         │
                  │       └──► Grafana   :3000              │
                  └─────────────────────────────────────────┘
                                      ▲
                           CronJob (backup every 30 min)
```

The pod contains three containers:
- **FastAPI** — main application
- **Prometheus** — metrics collection (scrape every 5s, 15-day retention, PersistentVolume)
- **Grafana** — visualization; datasource and dashboard are provisioned automatically via ConfigMap

---

## Quick Start / Быстрый старт

### Minikube (local)

```bash
# Install / upgrade
helm upgrade --install test ./helm/my-devops-app \
  -f helm/my-devops-app/values-minikube.yaml

# Remove
helm uninstall test
```

Add to `/etc/hosts`:
```
127.0.0.1  my-app.local grafana.local
```

### Azure AKS

**1. Provision infrastructure**

```bash
cd terraform
terraform init
terraform apply
```

Creates:
- Resource Group `rg-azure-test` (West Europe)
- AKS cluster `aks-azure-test` (1 × Standard_B2s, SystemAssigned identity, Azure CNI)

**2. Get kubeconfig**

```bash
terraform output -raw kube_config > ~/.kube/config-azure
export KUBECONFIG=~/.kube/config-azure
```

**3. Create GitHub Container Registry secret**

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<ghcr-token>
```

**4. Deploy the application**

```bash
helm upgrade --install prod ./helm/my-devops-app \
  -f helm/my-devops-app/values-azure.yaml
```

Available at:
- `http://fastapi.azure-test.local` — FastAPI
- `http://grafana.azure-test.local` — Grafana

---

## Project Structure / Структура проекта

```
.
├── terraform/
│   ├── provider.tf       # azurerm provider ~3.0
│   ├── main.tf           # Resource Group
│   ├── aks.tf            # AKS cluster
│   └── outputs.tf        # kube_config output
└── helm/my-devops-app/
    ├── Chart.yaml
    ├── values.yaml               # base values
    ├── values-minikube.yaml      # Minikube overrides
    ├── values-azure.yaml         # AKS overrides
    ├── dashboards/
    │   └── fastapi.json          # Grafana dashboard
    └── templates/
        ├── deployment.yaml       # Pod: FastAPI + Prometheus + Grafana
        ├── service.yaml
        ├── ingress.yaml
        ├── monitoring-configs.yaml   # ConfigMaps for Prometheus and Grafana
        ├── fastapi-dashboard-cm.yaml # ConfigMap with JSON dashboard
        ├── prometheus-pvc.yaml       # PersistentVolumeClaim
        └── backup-cronjob.yaml       # Prometheus backup every 30 min
```

---

## Requirements / Требования

- Terraform >= 1.3.0
- Helm 3
- Azure CLI (`az login`)
- kubectl
- Minikube (for local run)
