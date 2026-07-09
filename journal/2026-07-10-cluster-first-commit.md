# 2026-07-10 the cluster made its first commit

Enabling Flux image automation: when CI publishes a new
`dev-activity` image, the cluster itself commits the new tag into this repo
(`apps: auto-update images`, author fluxcdbot) and then deploys it. The full
loop is now code → CI → ghcr → bot commit → Flux → CronJob, with zero manual
steps — and every deploy still leaves a revertable commit in history.

The insight worth keeping: `:latest` + `imagePullPolicy: Always` was also
"automation", but it bypassed Git. Image automation restores the
automation *through* Git.

Two things had to change to make it work:

- The bootstrap deploy key was read-only; the automation controller must
  push. Re-added the same public key on GitHub with write access — the
  private half in the cluster never changed.
- Commit-SHA tags can't be ordered, so CI
  now adds a sortable tag `main-<unix-ts>-<sha>`; the ImagePolicy extract the timestamp and picks the numerical max.
