# CloudNativePG

PostgreSQL operator reference. Manifest: `infrastructure/postgres/cluster.yaml`.

## Status

```bash
kubectl get cluster -n homelab           # operator's view: READY, STATUS, PRIMARY
kubectl get pods -n homelab              # the actual instance pods
```

> Healthy = `Cluster in healthy state`. The `Cluster` type is a CRD (custom resourse definitions)—
> it only exists because the operator installed it.

## Services (created automatically)

| Service       | Meaning    | Routes to             | Use for            |
| ------------- | ---------- | --------------------- | ------------------ |
| `postgres-rw` | read-write | primary only          | apps that write    |
| `postgres-r`  | read       | any instance          | reads, any source  |
| `postgres-ro` | read-only  | replicas only         | heavy read offload |

> Apps should always connect via `postgres-rw:5432`. On failover the
> operator repoints the service — the app never notices.

## Credentials

```bash
# operator-generated: database `app`, user `app`, Secret `postgres-app`
kubectl get secret postgres-app -n homelab -o jsonpath='{.data.password}' | base64 -d; echo
kubectl get secret postgres-app -n homelab -o jsonpath='{.data.uri}' | base64 -d; echo
```

> The Secret also holds ready-made connection strings (`uri`, `jdbc-uri`).
> The postgres superuser is disabled for external access by default.

## psql inside the pod

```bash
kubectl exec -it postgres-1 -n homelab -- psql
```

```
\l    -- databases        \du   -- roles
\c app -- switch database \q    -- quit (commands start with \)
```

## Operator itself

```bash
kubectl get pods -n cnpg-system          # controller must be 1/1 Running
kubectl logs -n cnpg-system deploy/cnpg-controller-manager   # operator logs
```
