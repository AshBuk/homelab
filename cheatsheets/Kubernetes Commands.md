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

## Apply — "make the cluster match the YAML"

```bash
kubectl apply -f <file.yaml>             # create or update from manifest
kubectl apply --server-side -f <url>     # for operators with huge CRDs
kubectl get pods -n <ns> -w              # watch status changes live (Ctrl+C to stop)
```

> `-f` paths are relative to the current directory, not the repo root.
> `--server-side` is needed when objects exceed the 256KB annotation limit
> of client-side apply (CNPG, most operators).

## Inspect & debug — "why is it broken?"

```bash
kubectl describe pod <pod> -n <ns>   # full details: events, restarts, mounts
kubectl logs <pod> -n <ns>           # container stdout/stderr
kubectl logs <pod> -n <ns> -f        # follow live (like tail -f)
```

> Debug order: `get pods` (status) → `describe` (events at the bottom) →
> `logs` (the app itself). Most answers are in `describe` events.

## Exec — "get a shell inside a container"

```bash
kubectl exec -it <pod> -n <ns> -- <cmd>      # -it = interactive terminal
kubectl exec -it postgres-1 -n homelab -- psql   # psql inside the DB pod
```

> Multi-container pod? kubectl picks the default and says so;
> use `-c <container>` to choose explicitly.

## Secrets — "read a generated password"

```bash
kubectl get secret <name> -n <ns> -o jsonpath='{.data.password}' | base64 -d; echo
```

> Secrets are base64-encoded, not encrypted. `jsonpath` extracts one field.

## Resource types & shortnames

```bash
kubectl api-resources                        # all types + SHORTNAMES column
kubectl api-resources --api-group=<group>    # e.g. postgresql.cnpg.io (CRDs)
kubectl get svc,pvc,secret -n <ns>           # several types in one query
```

> Common shortnames: po=pods, ns=namespaces, svc=services,
> pvc=persistentvolumeclaims, deploy=deployments.

## Namespaces

```bash
kubectl get namespaces               # or get ns
kubectl create namespace <name>
```

## Services & storage

```bash
kubectl get svc -n kube-system   # Traefik's LoadBalancer + ports
kubectl get storageclass         # local-path should be (default)
```
