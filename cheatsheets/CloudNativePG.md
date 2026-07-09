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
kubectl exec -it postgres-1 -n homelab -- psql                    # superuser
kubectl exec -it postgres-1 -n homelab -- psql -U postgres -d app # straight into app db
```

```
\l    -- databases        \du   -- roles
\c app -- switch database \dt   -- tables in current db
\q    -- quit (commands start with \)
```

## Query jsonb payloads

```sql
SELECT DISTINCT jsonb_object_keys(payload) FROM github_events;  -- what keys exist
SELECT jsonb_pretty(payload) FROM github_events LIMIT 1;        -- formatted view
SELECT payload->>'ref' FROM github_events LIMIT 5;              -- ->> field as text
SELECT created_at::date AS day, count(*)                        -- ::date truncates,
FROM github_events GROUP BY day ORDER BY day DESC;              -- groups per day
```

> `->>` returns NULL for a missing key (no error) — an all-empty column
> means the key isn't in the data, not that the query is wrong.

## Operator itself

Managed by Flux HelmRelease `cnpg` (`infrastructure/cnpg/release.yaml`).
Upgrade = bump the chart `version:` there and push — never kubectl apply.

```bash
kubectl get pods -n cnpg-system          # controller must be 1/1 Running
kubectl logs -n cnpg-system deploy/cnpg-cloudnative-pg   # operator logs
flux get helmrelease cnpg -n cnpg-system # install/upgrade status
```
