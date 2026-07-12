# FluxCD

GitOps reference. Setup: bootstrapped to `clusters/homelab/`, Kustomizations
`infrastructure` and `apps` (in `clusters/homelab/*.yaml`), repo AshBuk/homelab.

## Status — the first command to run

```bash
flux get kustomizations                  # what's applied, from which revision
flux get sources git                     # is Git fetching OK (repo reachable, key valid)
flux get all                             # everything Flux manages, one screen
flux check                               # controllers healthy + versions
```

> Ready `True` + `Applied revision: main@sha1:...` = cluster matches that
> commit. Ready `False` — read the MESSAGE column first, it usually names
> the broken manifest.

## Force sync (don't wait for the interval)

```bash
flux reconcile source git flux-system            # fetch Git now
flux reconcile kustomization apps                # re-apply apps now
flux reconcile kustomization apps --with-source  # both in one go
```

> Normally not needed: a push lands within ~1m (GitRepository interval).
> Use after fixing a broken manifest to see the result immediately.

## Watch a rollout live

```bash
flux get kustomizations --watch          # statuses as they change (Ctrl+C to stop)
flux events                              # recent events, human-readable
flux events --for Kustomization/apps     # events for one object only
```

## Logs and debugging

```bash
flux logs                                # all controllers, merged
flux logs --level error                  # errors only
flux logs --kind Kustomization --name apps       # one object's log trail
kubectl get pods -n flux-system          # the controllers themselves
```

> Order of debugging: `flux get sources git` (can it fetch?) →
> `flux get kustomizations` (can it apply?) → `flux logs --level error`.

## Who manages this resource?

```bash
flux trace -n homelab cronjob/dev-activity   # object → Kustomization → Git commit
flux tree kustomization apps                 # everything a Kustomization owns
```

> `trace` answers "where did this come from" — shows the exact commit and
> the Kustomization that applied the object. Great for adopted resources.

## Helm releases (helm-controller)

```bash
flux get helmreleases -A                 # every release: revision, Ready, message
flux reconcile helmrelease cnpg -n cnpg-system --with-source   # re-sync now
flux logs --kind HelmRelease --name cnpg # install/upgrade log trail
kubectl describe helmrelease <name> -n <ns>   # Conditions + Helm log tail
```

> Upgrade = bump `version:` in the HelmRelease file, push. Rollback =
> git revert. The helm CLI is never needed — helm-controller runs the
> install/upgrade inside the cluster.

### Failed release stuck? ("Failed to upgrade after 1 attempt(s)")

```bash
flux suspend helmrelease <name> -n <ns>  # step 1: clears the failure state
flux resume helmrelease <name> -n <ns>   # step 2: fresh reconcile from zero
```

> By default a HelmRelease gets **zero retries**: one failed install/upgrade
> (e.g. an image pull slower than `timeout`) and helm-controller waits for
> a human — further reconciles refuse instantly. suspend/resume resets the
> counter. Prevention: `remediation.retries: 3` under both `install:` and
> `upgrade:` in the HelmRelease spec (see infrastructure/monitoring/release.yaml).

## Image automation (auto-deploy new images)

```bash
flux get images all                      # repository scan → chosen tag → last commit
flux reconcile image repository dev-activity   # scan ghcr now (after a CI build)
flux reconcile image update flux-system  # write the bot commit now
flux suspend image update flux-system    # freeze auto-deploys (e.g. bad release)
```

> The controller only rewrites lines marked `# {"$imagepolicy": "ns:name"}`
> under `update.path`. CI must produce sortable tags (`main-<ts>-<sha>`) —
> raw SHA tags have no order. Rollback: suspend, then git revert the bot
> commit.

## Suspend / resume

```bash
flux suspend kustomization apps          # stop reconciling (Git is ignored)
flux resume kustomization apps           # re-enable + reconcile immediately
```

> Suspend before manual experiments with `kubectl` — otherwise Flux reverts
> your changes on the next drift check. Resume restores Git as truth.

## Deleting things — two different intents

```bash
# Delete a resource FROM THE CLUSTER: remove its file from Git, push.
# prune: true does the rest. No kubectl delete.

kubectl delete kustomization apps -n flux-system   # removes apps AND everything
                                                   # it manages (prune) — rarely
                                                   # what you want
flux uninstall                           # removes Flux itself, keeps deployed
                                         # workloads running (unmanaged)
```

> With `prune: true` a deleted file = deleted resource. Deleting
> `infrastructure/postgres/cluster.yaml` deletes the database, PVC included.

## Disaster recovery

```bash
flux bootstrap github --owner=AshBuk --repository=homelab \
  --branch=main --path=clusters/homelab --personal
```

> Bootstrap is idempotent: on a fresh cluster it reinstalls Flux, which then
> restores everything else from Git. Needs a PAT (Contents RW +
> Administration RW for a fine-grained one); delete the token afterwards —
> steady-state access uses the SSH deploy key in Secret `flux-system/flux-system`.
