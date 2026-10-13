---
name: check-after-deploy
description: "Check production health after a deployment, scoped to the change shipped in this session. To decide whether a rolled-out ticket can be closed, use prod-acceptance."
---

# Check After Deploy

A feature or bugfix has been deployed to production. Check the logs, metrics, error tracker, and database (and anything else relevant) to surface any issues the release may have introduced, then give the user a report.

## Scope the check to what shipped

You are usually called at the end of the session that built and shipped the change, so you already know what went out: the diff you wrote, the PR you merged, the migration that went with it. **That change is the scope of the check** — check what it touches, and leave the rest of production alone.

The pull is to sweep everything instead: the error tracker's top issues, every dashboard, anything that looks wrong. That answers a different question. Any busy system carries a standing rate of failing jobs, slow queries and exceptions that predate today's release and will outlive it, and a report full of them is noise the user has to filter — with the one regression that actually *is* yours buried somewhere inside it.

So go the other way round. Turn the diff into a short list of surfaces before you go looking, then check each one:

- **Request paths** — the routes, controllers and actions it touches: error rate, status codes, latency.
- **Background work** — the jobs and consumers it touches or enqueues: failures, retries, queue depth.
- **Data** — whether the migration applied, whether the new column is being written, whether the rows the feature creates look right, whether the queries it added turn up as slow.
- **Outbound calls** — the services the new code talks to: timeouts, error responses, latency.
- **Its own names** — the error classes it raises, and the files, classes and methods it adds, searched for by name in logs and stack traces. This is the sharpest signal you have that a failure belongs to this release rather than to the background.
- **Its switches** — whether the feature flag, env var or setting the change depends on is really set the way you assume in production.

And check that the change *works*, not only that nothing caught fire around it. A bugfix is verified by going and looking for the bug — the failing case should now succeed. A feature is verified by its own traffic: the new path being exercised, the rows being written, the metric moving. A new code path with no traffic at all is a finding, not a clean result — it usually means the deploy didn't take, the flag is off, or nothing can reach it.

## Telling a regression from the background

"It's broken" and "the release broke it" are different claims, and only the second is what the user asked about. Before you report anything, establish that it is new: time-bound every query to the deploy, and compare against the same window before it. An exception that was firing at the same rate yesterday isn't yours — it may still deserve a mention, but not as a release regression. When the deploy rolled out gradually, compare against the instances, region or cohort that haven't got it yet.

This cuts both ways: a familiar-looking error class that has suddenly started carrying your new file in its stack trace *is* yours, however long it has existed.

## When you don't know what shipped

Sometimes the session carries no such context — a fresh session, or a user who just says production got deployed and asks you to look. Try to recover it cheaply first: what merged into the default branch recently, what the deploy or release record names, what the last few commits touched. Recovered context is context, and everything above applies to it unchanged.

Only when the release genuinely can't be pinned down do you check everything: sweep the error tracker, the dashboards and the logs broadly over the window since the deploy, still weighing what is new in that window above what has always been there. Say in the report that this is what you did and why, so the user can hand you the missing context and get the narrow check instead.

## The report

Lead with the verdict on the release: healthy or not, and doing what it was meant to do or not. Then the findings, each with what you saw, when it started, and what ties it to the change. Don't pad the report with everything that came back clean.

Do say what you *couldn't* check — no metrics on that path, no access to that dashboard, not enough traffic yet to tell — because an unchecked surface silently reads as a healthy one. And if you trip over something seriously wrong that has nothing to do with this release, give it a line of its own, clearly marked as unrelated, rather than swallowing it or filing it as a regression.
