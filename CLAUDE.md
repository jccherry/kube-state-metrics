# Claude Code Project Instructions

These instructions are mandatory for all Claude Code sessions working in this repository.

## Project Overview

This repo deploys [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) to a k3s homelab cluster using a custom Helm chart. kube-state-metrics generates Prometheus metrics about the state of Kubernetes objects (deployments, pods, nodes, etc.).

## Infrastructure

| Component | Details |
|-----------|---------|
| Cluster | 3-node k3s (1 master + 2 workers) |
| Ingress | Traefik (built into k3s) |
| DNS | Pi-hole, domain: `*.cherrykube.lan` |
| Registry | Local Docker registry on `cherrydocker01.lan:5000` |
| Jump host | `cherryjump01` — the only host with `helm` and `kubectl` |
| Infra repo | [jccherry/k3s-homelab](https://github.com/jccherry/k3s-homelab) |

## Repo Structure

```
deployment/helm/kube-state-metrics/   # Helm chart (all K8s manifests)
  Chart.yaml                          # Chart metadata, appVersion tracks upstream
  values.yaml                         # Default configuration
  templates/                          # K8s resource templates
.github/workflows/
  ci.yml                              # helm lint + helm template validation
  auto-tag.yml                        # Semver tagging on merge to main
docs/
  DEPLOYMENT.md                       # Operational guide
```

## Development Workflow

1. **No local helm/kubectl** — this Mac does not have helm or kubectl. All chart testing and deployment happens on `cherryjump01` via SSH.
2. **Branch off main** — use `feat/`, `fix/`, `refactor/`, `docs/`, `chore/` prefixes.
3. **CI validates charts** — GitHub Actions runs `helm lint` and `helm template` on PRs.
4. **Deploy from cherryjump01** — `ssh cherry@cherryjump01`, clone/pull the repo, run `helm install/upgrade`.

## Versioning

Follows [Semantic Versioning](https://semver.org/). Tags are bare numbers (e.g., `1.0.0`). Auto-tagged on merge to `main` via `.github/workflows/auto-tag.yml`. Apply `semver:major`, `semver:minor`, or `semver:patch` labels to PRs before merging.

## Key Decisions

- **Official image from `registry.k8s.io`** — k3s nodes have internet access. Can switch to local mirror via `image.repository` in values.yaml.
- **Own namespace `kube-state-metrics`** — matches homelab pattern of per-app namespaces.
- **ClusterIP service** — internal-only; Prometheus scrapes directly. Optional Traefik ingress available.
- **Homelab-sized resources** — 50m/64Mi requests, 200m/256Mi limits.
