# 2026-07-09 why dev-activity exists

GitHub only keeps a 90-day/300-event window of activity feed. The
`dev-activity` CronJob copies that feed into my own PostgreSQL every hour,
so the history accumulates forever and stays queryable in ways GitHub never
shows: hours of day, code vs review ratio, per-repo breakdown, streaks.

The design tolerates the server being offline: every run re-fetches the
whole available window and `ON CONFLICT DO NOTHING` skips what's already
stored, so gaps heal themselves. At my current pace (~130 events / 90 days)
even booting the server once a month loses nothing.

Caveat: the database lives on the old laptop's disk. Years
of history deserve a backup; CloudNativePG can ship backups to S3-compatible
storage, a good future phase.
