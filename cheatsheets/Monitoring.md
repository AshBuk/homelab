# Monitoring

kube-prometheus-stack reference. Setup: Flux HelmRelease in
`infrastructure/monitoring/`, namespace `monitoring`, chart pinned in Git.

## What the stack is — five components, one chart

| Component | Kind | Job |
|---|---|---|
| Prometheus Operator | Deployment | watches `ServiceMonitor`/`PodMonitor` CRDs, generates Prometheus config |
| Prometheus | StatefulSet (PVC) | the time-series database; scrapes `/metrics` endpoints (pull model) |
| Grafana | Deployment (PVC) | dashboards over Prometheus (and other datasources) |
| kube-state-metrics | Deployment | metrics about k8s objects: pod status, job success, replicas |
| node-exporter | DaemonSet | metrics about the machine: CPU, RAM, disk, network |

## Access — port-forward from the workstation

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# → http://localhost:3000  (login: admin; password changed from chart default in UI)

kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# → http://localhost:9090  (raw Prometheus UI)
```

> Tunnel lives while the command runs; Ctrl+C to stop

## Is scraping healthy?

- Prometheus UI → **Status → Targets**: every target `UP`. A `DOWN` target
  names the endpoint and the error (connection refused, timeout, 403).
- PromQL `up` — same info as a metric: `1` = scrape OK, `0` = failing.

```bash
kubectl get servicemonitors -n monitoring    # what the operator scrapes from
kubectl get prometheus -n monitoring         # the Prometheus CRD instance
```

> Scrape config is declarative: a `ServiceMonitor` selects Services by
> label and names a port; the operator turns it into Prometheus config.
> Our values set `serviceMonitorSelectorNilUsesHelmValues: false`, so
> monitors are discovered cluster-wide — needed for our own apps later.

## PromQL starters (Grafana → Explore, or Prometheus → Graph)

```promql
up                                            # every scrape target, 1/0
kube_pod_status_phase{namespace="homelab"}    # pod lifecycle as metrics
kube_job_status_succeeded                     # did the CronJob's last runs work
node_memory_MemAvailable_bytes                # free RAM on the node
rate(container_cpu_usage_seconds_total{namespace="homelab"}[5m])  # CPU by container
```

> `rate(x[5m])` = per-second growth of a counter over 5 minutes — counters
> only ever go up, `rate` turns them into "speed right now".

## k3s specifics (encoded in release.yaml values)

- `kubeEtcd`, `kubeScheduler`, `kubeControllerManager`, `kubeProxy`
  scraping **disabled**: k3s is a single binary with kine/SQLite instead
  of etcd — those targets don't exist, leaving them on = permanent DOWN
  targets + false alerts.
- Storage via `local-path`: Prometheus 10Gi PVC (`retention: 20d`),
  Grafana 2Gi PVC (hand-made dashboards + changed admin password live there).

## Lessons from the first install

- Big chart + slow network: the Grafana image (378MB) pulled longer than
  the Helm `timeout` — release went `False`, pods finished fine anyway.
  Fix was a reconcile after the pulls completed; prevention is
  `remediation.retries` in the HelmRelease (see FluxCD sheet).
- Liveness probes kill slow-starting containers on a loaded node: kubelet
  restarted Grafana twice during the image-pull storm. Restart counts
  right after an install are noise, not a problem — check `AGE` vs the
  restart timestamp.
