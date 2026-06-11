# Writeup

> Keep this brief — bullets are fine. We read it closely; it's where we see your judgment.
> Delete these prompts as you go. (See ASSIGNMENT §5.5 / §6.)

## How to run it
One command (`make run`, a script — whatever) that does EL → dbt end-to-end, plus anything else we
need to know to reproduce. Assume a clean checkout + `make up && make seed`.

## EL / CDC strategy
How you extract and load each source, and how the incremental loop stays **idempotent** and
captures **deletes** (hard-deleted users, churned accounts). What's your watermark / change-detection
approach? How do you avoid duplicating or losing rows across a `tick`?

## Data-quality issues you found
Which of the source's messy bits you hit (pagination, schema drift, out-of-order / duplicate /
missing / late usage files, orphan billing customers, plan price versions, …) and how you handled
each.

## Key modeling decisions
Staging → marts layering, grain, point-in-time subscription/plan state, annual→monthly normalization.

**MRR-movement convention (state it):** how you classify new / expansion / contraction / churn, and
your call on the ambiguous cases (e.g. plan downgrade at flat seats, reactivation after churn).
There's no single right answer — we grade the reasoning and consistency.

## Known gaps & bugs (be honest)
What's not working, what's unfinished, what you'd verify with more time. A clearly flagged bug +
how you'd fix it beats shipping it silently.

## With more time / productionizing
Orchestration, scheduling, alerting, tests, the parts you stubbed.

## How you used AI
2–3 sentences: where it helped, and at least one place where it was wrong, incomplete, or you
overrode it. In your own voice.
