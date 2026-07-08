# 2026-07-08 GitHub events API sends less data without a token

**Symptom.** Tried to count commits per day by summing the `distinct_size`
field from the stored event payloads. The whole column came back empty.

Empty here means NULL: the `->>` operator does not fail when a JSON key is
missing — it just returns NULL, and a sum of NULLs is NULL. So an empty
column is a signal: *this field does not exist in the data*.

**Cause.** Instead of guessing, asked the database what the payloads
actually contain:

```sql
SELECT DISTINCT jsonb_object_keys(payload)
FROM github_events WHERE type = 'PushEvent';
```

Only five fields: `push_id`, `ref`, `head`, `before`, `repository_id`. 
Reason: without a token, GitHub sends a
trimmed-down PushEvent. With a token it sends the full version, including
the `commits` array. Verified both with curl.

**Fix.**

1. Created a fine-grained personal access token (no extra permissions
   needed — just being authenticated is enough).
2. Put it in the `github-token` Secret; the CronJob already reads it.
3. Old rows stayed trimmed (inserts never overwrite existing rows), so ran
   `TRUNCATE github_events;` once. GitHub keeps 90 days of history, so the
   next run refilled everything — this time with full payloads.

**Lesson.** Storing the raw `jsonb` payload paid off. The data had a
different shape than expected, yet no Go code and no schema had to change:
both finding the problem and adapting to it were just SQL queries.
