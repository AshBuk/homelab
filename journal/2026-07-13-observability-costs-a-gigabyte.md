# 2026-07-13 observability costs a gigabyte

First look at the Grafana cluster dashboard. Monitoring itself uses
~949 MiB, while Postgres and dev-activity together use ~25 MiB. The
watcher is 40x heavier than the workload. Good thing we picked Flux over ArgoCD to save RAM for exactly this.

Also noted Memory Limits Commitment at 93%. Memory does not throttle
like CPU, it just runs out and something gets OOM-killed. Rule from now
on: every new workload ships with a memory request and limit.
