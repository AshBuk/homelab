# 2026-07-08 GitHub events API no longer includes commit details in PushEvent

**Symptom.** Tried to count commits per day by summing the `distinct_size`
field from the stored event payloads. The whole column came back empty.

Empty here means NULL: the `->>` operator does not fail when a JSON key is
missing — it just returns NULL, and a sum of NULLs is NULL. So an empty
column is a signal: *this field does not exist in the data*.

**Investigation.** Instead of guessing, asked the database what the
payloads actually contain:

```sql
SELECT DISTINCT jsonb_object_keys(payload)
FROM github_events WHERE type = 'PushEvent';
```

Only five fields: `push_id`, `ref`, `head`, `before`, `repository_id`.
No commit list, no counters.

First theory: authentication. Added a fine-grained token — and proved it
was *delivered* (the `x-ratelimit-limit` response header jumped from 60 to
5000, i.e. the request was authenticated) but *didn't help*
(`grep -c distinct_size` over the response = 0). Ruling the token out led
to the real cause.

**Cause.** GitHub [changed the Events API](https://github.blog/changelog/2025-08-08-upcoming-changes-to-github-events-api-payloads/)
on October 7, 2025: commit summaries and counts were removed from PushEvent
payloads *for everyone*, token or not, to make the feed faster. Only
identifiers remain. This is the API's permanent shape now.

**Fix / decisions.**

- Kept the token anyway: 5000 requests/hour instead of 60, and private
  events appear in the feed.
- "Commits per day" becomes "pushes per day" — `count(*)` over PushEvent
  rows, an honest activity proxy available from the data we have.
- Exact per-day commit counts, if ever needed, should come from the
  GraphQL `contributionsCollection` (the profile contribution calendar) —
  a separate collector query, not from REST events.

**Lessons.**

- Debug by elimination with hard evidence: `x-ratelimit-limit` proved the
  token worked, which is exactly what pointed away from it.
- APIs you don't control change shape under you. Storing the raw `jsonb`
  payload meant nothing broke when it happened — diagnosing and adapting
  were SQL queries, not code changes.
