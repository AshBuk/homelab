# Kubernetes Commands

Working kubectl reference 

> `-A` = all namespaces, `-n <ns>` = one namespace. Without either, kubectl
> looks only in `default` — the #1 reason something "isn't there".

## Cluster & nodes — "is the cluster alive?"

```bash
kubectl get nodes                # node list; STATUS must be Ready
kubectl top nodes                # CPU/RAM per node (via metrics-server)
kubectl cluster-info             # API server & CoreDNS endpoints
```

## Pods — "what's running?"

```bash
kubectl get pods -A              # everything in the cluster
kubectl get pods -n homelab      # only our namespace
kubectl top pods -A              # CPU/RAM per pod
```

## Inspect & debug — "why is it broken?"

```bash
kubectl describe pod <pod> -n <ns>   # full details: events, restarts, mounts
kubectl logs <pod> -n <ns>           # container stdout/stderr
kubectl logs <pod> -n <ns> -f        # follow live (like tail -f)
```

> Debug order: `get pods` (status) → `describe` (events at the bottom) →
> `logs` (the app itself). Most answers are in `describe` events.

## Namespaces

```bash
kubectl get namespaces
kubectl create namespace <name>
```

## Services & storage

```bash
kubectl get svc -n kube-system   # Traefik's LoadBalancer + ports
kubectl get storageclass         # local-path should be (default)
```
