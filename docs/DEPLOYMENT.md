# Deployment Guide

## Prerequisites

- SSH access to `cherryjump01` (jump host with `helm` and `kubectl`)
- k3s cluster running (managed by [jccherry/k3s-homelab](https://github.com/jccherry/k3s-homelab))

## Install

```bash
ssh cherry@cherryjump01

git clone git@github.com:jccherry/kube-state-metrics.git ~/kube-state-metrics
cd ~/kube-state-metrics

helm install kube-state-metrics ./deployment/helm/kube-state-metrics \
  --namespace kube-state-metrics \
  --create-namespace
```

## Upgrade

```bash
ssh cherry@cherryjump01
cd ~/kube-state-metrics && git pull

helm upgrade kube-state-metrics ./deployment/helm/kube-state-metrics \
  --namespace kube-state-metrics
```

## Uninstall

```bash
helm uninstall kube-state-metrics --namespace kube-state-metrics
kubectl delete namespace kube-state-metrics
```

## Rollback

```bash
# List release history
helm history kube-state-metrics --namespace kube-state-metrics

# Rollback to a specific revision
helm rollback kube-state-metrics <REVISION> --namespace kube-state-metrics
```

## Configuration Values

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `registry.k8s.io/kube-state-metrics/kube-state-metrics` | Container image. Change to local mirror if needed. |
| `image.tag` | `""` | Overrides `appVersion` from Chart.yaml |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `replicas` | `1` | Number of replicas |
| `serviceAccount.create` | `true` | Create a dedicated ServiceAccount |
| `serviceAccount.name` | `""` | Override SA name (default: release fullname) |
| `service.type` | `ClusterIP` | Service type |
| `service.metricsPort` | `8080` | Metrics port |
| `service.telemetryPort` | `8081` | Telemetry/self-metrics port |
| `ingress.enabled` | `false` | Enable Traefik ingress |
| `ingress.className` | `traefik` | Ingress class |
| `ingress.host` | `kube-state-metrics.cherrykube.lan` | Ingress hostname |
| `resources.requests.cpu` | `50m` | CPU request |
| `resources.requests.memory` | `64Mi` | Memory request |
| `resources.limits.cpu` | `200m` | CPU limit |
| `resources.limits.memory` | `256Mi` | Memory limit |
| `collectors` | `[]` | Restrict collectors (empty = all defaults) |
| `namespaces` | `[]` | Restrict namespaces (empty = all) |

### Override Example

```bash
helm install kube-state-metrics ./deployment/helm/kube-state-metrics \
  --namespace kube-state-metrics \
  --create-namespace \
  --set ingress.enabled=true \
  --set resources.limits.memory=512Mi
```

## RBAC

The chart creates a **ClusterRole** with read-only (`list`, `watch`) access to standard Kubernetes resource types across all API groups:

- **Core**: pods, services, nodes, configmaps, secrets, namespaces, endpoints, PVs, PVCs, etc.
- **Apps**: deployments, daemonsets, statefulsets, replicasets
- **Batch**: jobs, cronjobs
- **Networking**: ingresses, networkpolicies
- **Storage**: storageclasses, volumeattachments
- **RBAC**: roles, clusterroles, bindings
- **Others**: HPAs, PDBs, leases, CSRs, webhooks

A **ClusterRoleBinding** binds this role to the chart's ServiceAccount. This is cluster-scoped because kube-state-metrics needs visibility into all namespaces.

## Prometheus Integration

Add this scrape config to your Prometheus configuration:

```yaml
scrape_configs:
  - job_name: kube-state-metrics
    static_configs:
      - targets:
          - kube-state-metrics.kube-state-metrics.svc.cluster.local:8080
```

Or if using Prometheus Operator / ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-state-metrics
  namespace: kube-state-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  endpoints:
    - port: metrics
```

## Grafana Dashboards

### Import via UI

1. Open Grafana (`http://cherrygraf01:3000`)
2. Go to **Dashboards** > **New** > **Import**
3. Upload or paste the JSON from any file in `dashboards/`
4. Select a Prometheus data source when prompted

### Import via API

```bash
# Create a folder for the dashboards
curl -X POST http://cherrygraf01:3000/api/folders \
  -H "Content-Type: application/json" \
  -d '{"title": "kube-state-metrics", "uid": "kube-state-metrics"}'

# Import each dashboard
for f in dashboards/*.json; do
  curl -X POST http://cherrygraf01:3000/api/dashboards/db \
    -H "Content-Type: application/json" \
    -d "{\"dashboard\": $(cat "$f"), \"folderUid\": \"kube-state-metrics\", \"overwrite\": true}"
done
```

### Kubelet/cAdvisor Scraping (for CPU/Memory Usage Panels)

The Resource Usage, Cluster Overview, and Node Overview dashboards require actual CPU and memory metrics from kubelet/cAdvisor. These metrics (`container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`) are not provided by kube-state-metrics itself.

**1. Create RBAC on k3s (from cherryjump01):**

```bash
kubectl create namespace prometheus-external
kubectl create serviceaccount prometheus-scraper -n prometheus-external
kubectl create clusterrole prometheus-scraper \
  --verb=get,list,watch \
  --resource=nodes,nodes/proxy,nodes/metrics,nodes/stats,services,endpoints,pods
kubectl create clusterrolebinding prometheus-scraper \
  --clusterrole=prometheus-scraper \
  --serviceaccount=prometheus-external:prometheus-scraper
```

**2. Generate a bearer token:**

```bash
kubectl create token prometheus-scraper -n prometheus-external --duration=8760h
```

**3. Save the token on the Prometheus host:**

Write the token to `/etc/prometheus/k3s-bearer-token`.

**4. Add scrape configs to `prometheus.yml`:**

```yaml
scrape_configs:
  - job_name: "k3s-kubelet-cadvisor"
    metrics_path: "/metrics/cadvisor"
    scheme: "https"
    bearer_token_file: "/etc/prometheus/k3s-bearer-token"
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets:
          - "192.168.88.201:10250"
          - "192.168.88.202:10250"
          - "192.168.88.203:10250"

  - job_name: "k3s-kubelet"
    metrics_path: "/metrics"
    scheme: "https"
    bearer_token_file: "/etc/prometheus/k3s-bearer-token"
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets:
          - "192.168.88.201:10250"
          - "192.168.88.202:10250"
          - "192.168.88.203:10250"
```

**5. Validate and reload:**

```bash
promtool check config /etc/prometheus/prometheus.yml
rc-service prometheus reload
```

## Verification

```bash
# Pod should be Running
kubectl get pods -n kube-state-metrics

# Service should have ClusterIP with ports 8080, 8081
kubectl get svc -n kube-state-metrics

# RBAC should be created
kubectl get clusterrole,clusterrolebinding | grep kube-state-metrics

# Metrics should return kube_* metrics
kubectl port-forward svc/kube-state-metrics 8080:8080 -n kube-state-metrics &
curl http://localhost:8080/metrics | head -20

# Logs should show no errors
kubectl logs deployment/kube-state-metrics -n kube-state-metrics
```
