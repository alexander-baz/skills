---
name: prod-acceptance
description: "Accept a shipped ticket on production: prove with before/after numbers that the change works for real users, find regressions and unfinished parts, then report and post the verdict to the ticket."
---

# Production Acceptance

A ticket's change has been rolled out to production, and the user wants to know whether it can be closed. Answer three questions with numbers from the source of truth: when exactly it went live, whether it does what it was meant to do for real users, and what it broke or left undone. Then report, and with the user's approval post the verdict to the ticket.

This is narrower than a general health check and stricter than one. A health check asks whether anything caught fire; acceptance asks whether *this* change works, and "nothing caught fire" is not an answer to that.

The steps below are the method. Where things are in a given project (console access, rollout markers, data pitfalls, how to post to the tracker) comes from the repository's and the user's own instructions and memory; load the relevant parts before the first query.

## Start from what shipped, not from the ticket

First look for an earlier verification of the same ticket in memory. If there is one, start from its conclusions and its list of paths not yet seen live, and spend the session on those rather than recomputing what is already settled.

Then read the ticket (acceptance criteria, comments) and every MR that carried it — tasks often ship as several MRs across services plus follow-up fixes. **Verify what the code changed, not what the ticket says.** The diff and the MR description (risks, out of scope, follow-ups) define the mechanics you are looking for. Where the ticket asked for something the MRs deliberately didn't do, that is its own item in the report, not a failure of the change.

Before the first query, write down in chat:

- **The mechanics** — which code path changes and what trace it leaves: a status transition, a stored field, a log or event row, an alert, a metric, an analytics event, an error-tracker group.
- **The positive marker** — what the new branch firing looks like on one concrete object.
- **The effect** — which share, count or latency should move, and in which direction. Normalise by volume; absolute numbers move with traffic.
- **The regression markers** — what nearby must stay put: conversion of the affected method, neighbouring statuses, new error groups, validation failures.
- **The traps** — ways the direct metric can lie: an attribute linked only after the fact, sticky routing, an A/B split mistaken for the change, a gate the data bypasses by design. If the obvious metric can move for a reason unrelated to the fix, pick another one or explain the gap.

Writing this down first keeps the queries pointed at the change instead of at whatever looks odd in production.

## Pin down the rollout moment

The merge date is not the rollout, and neither is a master pipeline. Use the hardest marker the system offers: the after-deploy job's own report, the image tag the pods are running, a release tag in the error tracker, the timestamp of the shipped asset. When there is no direct marker, find the step in a metric the change moves — a sharp change in request rate per action is the rollout.

The last case of the old behaviour and the first case of the new one bracket it well. The earliest `created_at` among rows carrying the new marker does not: work started before the deploy finishes after it and drags the apparent moment back by days.

Check, too, that the rollout switched anything on. A change can ship dormant behind a flag, a per-account mapping, a rule, or a production config override that differs from the repo. Read the live values before assuming the feature is active.

## Measure before against after

Compare **equal windows over the same hours of the day and days of the week**, anchored at the rollout moment — Thursday 11:00 to Sunday 11:00 against the same shape a week earlier. The rollout day as a whole mixes both states and belongs to neither side. Drop windows that external events distort: month-start payday spikes, provider outages, rule edits in the admin made inside the window, other rollouts landing alongside.

**Every filtered count needs its denominator.** Zero matches after the rollout means nothing until you know how many rows the window had without the target predicate; a filter that never matches also returns zero. Better still, run the detector on the before window first: if it finds the cases the ticket was filed about and finds none after, the zero is real. If the event is rare, a handful a month, before/after can't prove anything — say so rather than reading a trend into three data points.

Fresh windows contain objects that haven't finished: orders and payments still waiting on a provider or a timeout will settle later, so success rates for the last day undercount. Count over the database, not from a few log samples, which can be off by an order of magnitude. For frontend changes, normalise error-tracker events by request volume, use percentiles for speed, and use a platform the change didn't touch as the control group.

## Find it on live objects

Aggregates aren't enough. Pick **two to five concrete objects** that went through the new branch and follow each through its full chain: status transitions, comments, event log, provider data, the analytics mirror. Pick one more object on the opposite branch, where the change must correctly *not* fire, and check that it didn't.

When the new branch hasn't run live yet — the triggering event simply hasn't happened since the rollout — that is a finding, not a pass. State it as "path X not seen live: 0 cases in N days against a background of M", and where you can, exercise the logic in memory on an old object without writing anything.

## Sort what you find

Every anomaly goes into exactly one bucket:

- **Regression** — appeared with the rollout and traces to the diff.
- **Incomplete fix** — a gap in the same mechanics the change addressed.
- **Pre-existing** — present before the rollout at the same rate.
- **Unrelated** — another rollout, an external outage, stage traffic leaking into production tooling, a rollout-window artefact.

Before naming a cause, walk the code from the entry point (the action, webhook or job) to the write. A hypothesis that a neighbouring ticket is to blame needs the same walk; the obvious suspect is often wrong. Anything involving money — a stuck refund, a wrong balance, a double charge — gets its own line with the amount, the user and the order.

## Stay read-only

Everything here reads. Run production console scripts inside a read-only transaction, set session parameters only with `SET LOCAL` inside it, and never with a plain `SET` through a pooled connection, where it outlives your session and reaches the application. Print after every step with its own rescue, so one failure doesn't swallow the output of the steps before it, and check column and table names against the code first — errors from production consoles land in the error tracker. Writes, backfills and setting changes go to the user as a command to run themselves. Never print secrets.

## The report

Lead with the verdict — works, works partially, or doesn't — and the rollout moment with how you determined it. Then, in the user's language and densely:

- **Mechanics** — the positive marker, how many cases, an example id.
- **Effect** — the metric before → after, with the windows and the denominator.
- **Regressions** — what you checked and what came back.
- **Not seen live** — which branches, and why.
- **Leftovers** — incomplete parts, follow-ups, manual clean-up.

Don't pad it with what came back clean, but do name what you couldn't check: an unchecked surface silently reads as a healthy one.

Post the verdict to the ticket **only after the user approves the text**. Propose follow-up tickets as text and create them only when asked; leave status transitions to the user.

Finally, save the verification to memory: the date, the rollout moment and how it was found, the key numbers, what isn't seen live yet, where the leftovers went, and **how to count** — indexes, filters, traps — so the next check of the same ticket starts there instead of from zero. A new pitfall that applies beyond this ticket goes where the project's general notes live, not into the ticket's note.
