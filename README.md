# Homelab

Bare-metal Kubernetes homelab — k3s cluster on dedicated hardware, running self-written Go microservices with PostgreSQL. Full cloud-native pipeline: Go code → multi-stage Docker build → GitHub Actions CI → GHCR → GitOps delivery via FluxCD/ArgoCD → Traefik Ingress routing → Prometheus metrics collection → Grafana observability dashboards. Infrastructure managed declaratively: all manifests version-controlled in Git, secrets encrypted with SOPS + age, zero manual deployments.
