# kube-state-metrics

Deploys [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) to a k3s homelab cluster via a custom Helm chart. Exposes Kubernetes object metrics for Prometheus scraping.

## Quick Start

From **cherryjump01** (the only host with `helm` and `kubectl` access):

```bash
git clone git@github.com:jccherry/kube-state-metrics.git ~/kube-state-metrics
cd ~/kube-state-metrics

helm install kube-state-metrics ./deployment/helm/kube-state-metrics \
  --namespace kube-state-metrics \
  --create-namespace
```

## Verify

```bash
kubectl get pods -n kube-state-metrics
kubectl get svc -n kube-state-metrics

# Test metrics endpoint
kubectl port-forward svc/kube-state-metrics 8080:8080 -n kube-state-metrics &
curl http://localhost:8080/metrics | head -20
```

## Upgrade

```bash
cd ~/kube-state-metrics && git pull
helm upgrade kube-state-metrics ./deployment/helm/kube-state-metrics \
  --namespace kube-state-metrics
```

## Uninstall

```bash
helm uninstall kube-state-metrics --namespace kube-state-metrics
kubectl delete namespace kube-state-metrics
```

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  k3s Cluster                     │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │  Namespace: kube-state-metrics            │   │
│  │                                           │   │
│  │  ┌─────────────┐    ┌─────────────────┐  │   │
│  │  │ Deployment  │    │ Service (ClusterIP)│ │   │
│  │  │ kube-state- │◄───│  :8080 (metrics) │  │   │
│  │  │  metrics    │    │  :8081 (telemetry)│  │   │
│  │  └──────┬──────┘    └─────────────────┘  │   │
│  │         │                                 │   │
│  │  ┌──────┴──────┐                         │   │
│  │  │ ServiceAcct │                         │   │
│  │  └──────┬──────┘                         │   │
│  └─────────┼─────────────────────────────────┘   │
│            │                                      │
│  ┌─────────┴──────────────────────────────────┐  │
│  │ ClusterRole + ClusterRoleBinding            │  │
│  │ (read-only access to all K8s resources)     │  │
│  └─────────────────────────────────────────────┘  │
│                                                    │
│  Prometheus ──scrape──► :8080/metrics              │
└────────────────────────────────────────────────────┘
```

## Configuration

Key values in `deployment/helm/kube-state-metrics/values.yaml`:

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `registry.k8s.io/kube-state-metrics/kube-state-metrics` | Container image |
| `image.tag` | `""` (uses appVersion) | Image tag override |
| `service.type` | `ClusterIP` | Service type |
| `resources.requests.cpu` | `50m` | CPU request |
| `resources.requests.memory` | `64Mi` | Memory request |
| `ingress.enabled` | `false` | Enable Traefik ingress |
| `collectors` | `[]` (all) | Restrict to specific collectors |
| `namespaces` | `[]` (all) | Restrict to specific namespaces |

See `docs/DEPLOYMENT.md` for the full operational guide.

## Grafana Dashboards

Pre-built dashboards live in `dashboards/` and can be imported into Grafana:

| Dashboard | File | Description |
|-----------|------|-------------|
| Resource Usage | `dashboards/resource-usage.json` | CPU & memory usage vs requests/limits per namespace, pod, container |
| Cluster Overview | `dashboards/cluster-overview.json` | High-level cluster stats, pod distribution, node status |
| Workload Status | `dashboards/workload-status.json` | Deployment, DaemonSet, StatefulSet, Job health |
| Node Overview | `dashboards/node-overview.json` | Per-node resource usage, capacity planning |

All dashboards use a `$datasource` template variable for Prometheus data source selection. See `docs/DEPLOYMENT.md` for import instructions.

## CI

GitHub Actions runs `helm lint` and `helm template` on every push and PR to `main`.
