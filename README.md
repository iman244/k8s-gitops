# ctesiphon-k8s-gitops

GitOps repository for the **Ctesiphon** homelab Kubernetes cluster — a single-node setup running Ubuntu 24.04 and Kubernetes v1.35. All cluster state is managed declaratively via ArgoCD, with Helm charts vendored in-repo and configuration split into a separate `config/` layer.

---

## Table of Contents

- [Cluster Overview](#cluster-overview)
- [Repository Structure](#repository-structure)
- [Bootstrap](#bootstrap)
- [Sync Wave Order](#sync-wave-order)
- [Applications](#applications)
  - [Infrastructure Layer](#infrastructure-layer)
  - [Workload Layer](#workload-layer)
  - [Observability Layer](#observability-layer)
- [Networking](#networking)
- [Image Auto-Updates](#image-auto-updates)
- [Secrets](#secrets)
- [Exposed Services](#exposed-services)

---

## Cluster Overview

| Property | Value |
|---|---|
| Node | Single-node (control-plane + workload) |
| OS | Ubuntu 24.04 |
| Kubernetes | v1.35.0 |
| CNI | Calico (`10.10.0.0/16`) |
| GitOps engine | ArgoCD |
| Ingress | Traefik v3 via Gateway API |
| Load balancer | MetalLB (L2 mode) |
| TLS | cert-manager (Let's Encrypt) |
| Observability | Victoria Metrics stack + Grafana |
| CD automation | ArgoCD Image Updater |

---

## Repository Structure

```
.
├── bootstrap/
│   └── root-app.yaml          # One-time bootstrap: points ArgoCD at apps/
├── apps/                      # ArgoCD Application manifests (app-of-apps)
│   ├── argocd-config/
│   ├── argocd-image-updater/
│   ├── cert-manager/
│   ├── gateway-api/
│   ├── metallb/
│   ├── metallb-config/
│   ├── task-manager/
│   ├── traefik/
│   ├── traefik-config/
│   ├── victoria-metrics-stack/
│   └── victoria-metrics-stack-config/
├── charts/                    # Vendored Helm charts (source of truth)
│   ├── argocd-image-updater/
│   ├── cert-manager/
│   ├── gateway-api/
│   ├── metallb/
│   ├── task-manager/          # Custom chart for the task manager app
│   ├── traefik/
│   └── victoria-metrics-k8s-stack/
└── config/                    # Environment-specific values and config objects
    ├── argocd/
    ├── argocd-image-updater/
    ├── metallb/
    ├── task-manager/
    ├── traefik/
    └── victoria-metrics-k8s-stack/
```

**Design pattern:** Each application has two ArgoCD `Application` objects — one pointing at `charts/<name>` (the Helm chart) and one pointing at `config/<name>` (live config objects like `IPAddressPool`, `Gateway`, `HTTPRoute`, etc.). This keeps the chart generic and the environment config independently versioned.

---

## Bootstrap

The cluster is bootstrapped once with:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

This creates the `root` ArgoCD Application, which recursively watches `apps/` and creates all child Applications. From that point on, every change goes through git.

---

## Sync Wave Order

ArgoCD sync waves control the rollout sequence to respect dependencies:

| Wave | Applications |
|---|---|
| 0 | `gateway-api`, `cert-manager`, `metallb` |
| 1 | `traefik`, `metallb-config` |
| 2 | `traefik-config`, `argocd-config`, `argocd-image-updater-config`, `victoria-metrics-stack` |
| 3 | `task-manager`, `victoria-metrics-stack-config` |

CRDs and load balancer infrastructure come up first, then ingress, then config that depends on those resources, then workloads.

---

## Applications

### Infrastructure Layer

#### MetalLB
- **Chart:** `charts/metallb`
- **Namespace:** `metallb-system`
- **Config:** `config/metallb/`
- Provides L2 load balancer. Allocated IP pool: `87.248.156.185/32` (single public IP).
- Both `controller` and `speaker` have `control-plane` toleration for the single-node setup.

#### Gateway API CRDs
- **Chart:** `charts/gateway-api/gateway-api.yaml`
- **Namespace:** `kube-system`
- Installed via `ServerSideApply=true` to handle large CRD objects.

#### Traefik
- **Chart:** `charts/traefik`
- **Namespace:** `traefik`
- **Config:** `config/traefik/`
- Runs as a single replica Deployment with `LoadBalancerIP: 87.248.156.185`.
- Uses the **Gateway API provider** (`kubernetesGateway: enabled: true`); Traefik's built-in Gateway resource is disabled (managed separately in `config/traefik/gateway.yaml`).
- Exposes ports 80 (HTTP) and 443 (HTTPS).

#### cert-manager
- **Chart:** `charts/cert-manager`
- **Namespace:** `cert-manager`
- Gateway API integration enabled (`enableGatewayAPI: true`).
- Issues Let's Encrypt certificates referenced by Gateway listeners.
- ClusterIssuer defined in `apps/cert-manager/clusterissuer.yaml`.

---

### Workload Layer

#### Task Manager
- **Chart:** `charts/task-manager`
- **Namespace:** `task-manager`
- **Values:** `config/task-manager/values.yaml` (overrides chart defaults; this is the file written to by Image Updater)

A full-stack application consisting of four components:

| Component | Image | Current Tag |
|---|---|---|
| Backend | `iman244/task-manager-fastapi` | `v1.0.1` |
| Frontend | `iman244/task-manager-nextjs` | `v1.0.4` |
| PostgreSQL | `postgres` | `18` |
| Redis | `redis` | `8.8` |

**Resource allocations (from `config/task-manager/values.yaml`):**

| Component | CPU req/limit | Memory req/limit |
|---|---|---|
| Backend | 15m / 150m | 210Mi / 320Mi |
| Frontend | 25m / 150m | 280Mi / 400Mi |
| PostgreSQL | 50m / 200m | 180Mi / 300Mi |
| Redis | 20m / 100m | 64Mi / 128Mi |

Both backend and frontend run 2 replicas. PostgreSQL data is persisted to hostPath `/kubernetes-volumes/task-manager/postgres` via a manually provisioned PV (`storageClassName: manual`).

Routing is via `HTTPRoute` resources (Gateway API) referencing the `traefik-gateway` in the `traefik` namespace, with a `ReferenceGrant` to allow cross-namespace binding.

---

### Observability Layer

#### Victoria Metrics Stack
- **Chart:** `charts/victoria-metrics-k8s-stack`
- **Namespace:** `victoria-metrics-stack`
- **Values:** `config/victoria-metrics-k8s-stack/values.yaml`

Includes: VMSingle (metrics storage), VMAgent (scraping), VMAlert, VMAlertmanager, Grafana, kube-state-metrics, prometheus-node-exporter.

Key configuration:
- Grafana admin credentials from existing secret `grafana-admin-secret`.
- Grafana persistence via existing PVC `grafana-pvc` (hostPath-backed PV defined in `config/victoria-metrics-k8s-stack/grafana-pv.yaml`).
- VMSingle storage: 2Gi PV defined in `config/victoria-metrics-k8s-stack/vmsingle-pv.yaml`.
- ArgoCD `ignoreDifferences` configured for webhook CA bundles and Grafana admin password to prevent sync loops from operator-managed fields.
- `ServerSideApply=true` required due to large CRD field managers.

---

## Networking

All external traffic enters through a single `Gateway` resource:

```
Internet → 87.248.156.185 (MetalLB) → Traefik Pod
           → HTTPRoute → Service → Pod(s)
```

The Gateway (`config/traefik/gateway.yaml`) has a single HTTP listener on port 80 accepting routes from all namespaces. HTTPS listeners with per-domain cert-manager certificates are defined inline on the Gateway.

ArgoCD itself is exposed via `config/traefik/httproute-argocd.yaml`.

### Exposed Hostnames

| Service | Hostname |
|---|---|
| Task Manager frontend | `task-manager.golden-horde.ir` |
| Task Manager backend | `task-manager-backend.golden-horde.ir` |
| Grafana | `grafana.golden-horde.ir` |
| ArgoCD | (defined in `config/traefik/httproute-argocd.yaml`) |

Cross-namespace routing (e.g., `task-manager` namespace → `traefik` namespace gateway) is permitted via `ReferenceGrant` objects.

---

## Image Auto-Updates

ArgoCD Image Updater watches for new semver tags on the task manager images and writes updated tags back to `config/task-manager/values.yaml` in the `main` branch.

**Configuration:** `config/argocd-image-updater/imageupdater.yaml`

```
ImageUpdater CR: task-manager
  ├── backend:  iman244/task-manager-fastapi  (strategy: semver, pattern: ^v\d+\.\d+\.\d+$)
  │   └── writes → backend.image.tag in config/task-manager/values.yaml
  └── frontend: iman244/task-manager-nextjs   (strategy: semver, pattern: ^v\d+\.\d+\.\d+$)
      └── writes → frontend.image.tag in config/task-manager/values.yaml
```

Write-back uses git push via secret `argocd/argocd-image-updater-secret`, targeting branch `main`.

---

## Secrets

Secrets are **not** stored in this repository. They are pre-provisioned on the cluster as Kubernetes Secrets and referenced by name:

| Secret Name | Namespace | Used By |
|---|---|---|
| `task-manager-secrets-external` | `task-manager` | Backend env vars (DB credentials, etc.) |
| `grafana-admin-secret` | `victoria-metrics-stack` | Grafana admin login |
| `argocd-image-updater-secret` | `argocd` | Image Updater git write-back |

---

## Day-2 Operations

### Applying a change
Push to `main`. ArgoCD auto-syncs (`automated: prune: true, selfHeal: true`) within the default poll interval.

### Forcing a sync
```bash
argocd app sync <app-name>
```

### Checking Image Updater status
```bash
kubectl logs -n argocd deployment/argocd-image-updater -f
kubectl get imageupdater -n argocd task-manager -o yaml
```

### Adding a new application
1. Vendor the chart into `charts/<name>/`
2. Add environment config to `config/<name>/`
3. Create `apps/<name>/application.yaml` with the appropriate sync wave annotation
4. Push — the `root` app picks it up automatically