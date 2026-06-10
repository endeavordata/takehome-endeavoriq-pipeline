# Data Engineer — Take-Home Assignment (Pipeline Build)

Welcome, and thanks for taking the time. This exercise gives us a realistic look at how you
build a data pipeline: pulling from messy, heterogeneous source systems, loading
incrementally, modeling with dbt, and reasoning about data quality.

**Plan for about 5 hours.** You may spend more if you want, but please don't feel obligated.
AI assistance is encouraged (see "Using AI" below), so the plumbing can go quickly — we'd rather
you spend your time on EL/CDC correctness, modeling judgment, and sanity-checking outputs than
grinding through boilerplate. We would much rather see a clean, correct *slice* done well than
everything done sloppily — **partial submissions are explicitly fine.** For anything you don't get
to, a short note on how you'd approach it is worth a lot. Finishing every part is not the bar.

Your submission is **code, not data**: we evaluate by running your pipeline ourselves against
a fresh copy of the environment. So make it reproducible — and make it work you can explain.

If something is genuinely blocking you (a container won't start, a question is ambiguous in a
way you can't resolve) — email us. Don't burn an hour stuck.

---

## Using AI on this assignment

**AI assistance is strongly encouraged** — we use Claude every day at Endeavor Labs, and we want
to see how you work *with* it. We recommend the **Claude Code CLI**, and we've included a Claude
credit with this assignment to cover it. You're welcome to use other tools (Cursor, ChatGPT, etc.)
if you prefer — use whatever makes you effective.

Here's the important part: Claude can produce most of this code quickly. **That's fine — and it's
exactly why the bar isn't the output.** In a follow-up conversation we'll walk through your
submission together: why you modeled it the way you did, how you validated correctness, how your
CDC handles deletes — and we'll likely ask you to extend the pipeline or write a model live. Work
you can't explain won't survive that conversation. So treat the AI as a fast pair-programmer, not
an oracle: read what it gives you, check it against the data, and make the decisions your own.
"hey Claude, do this assignment" is not the assignment.

To get started with Claude Code, install it and run `claude` from the repo root:

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash
# or via npm: npm install -g @anthropic-ai/claude-code
```

On first run it'll prompt you to sign in. Claude Code runs in your terminal and can drive Docker,
psql, and your EL/dbt code directly. See [code.claude.com/docs](https://code.claude.com/docs) for
setup and docs.

---

## 1. Setup (~10 minutes)

### What you need
- **Docker** (Desktop or Engine) with Compose v2.
- **A SQL editor / DuckDB CLI** if you like (optional).
- Whatever you build your EL in — Python, [dlt](https://dlthub.com/), etc. — your choice.

### Start the source systems
```bash
make up      # starts Postgres + the billing API
make seed    # generates ~18 months of history across all three sources
```
The first run pulls the billing-API and simulator images from `ghcr.io/endeavordata/…` (public —
no login required; just have Docker running). You should now have three live sources (see §3).
Verify:
```bash
docker compose exec app-db psql -U endeavoriq -c "select count(*) from app.accounts;"   # ~500
curl -s localhost:8080/v1/invoices | head -c 400                                          # JSON
ls drop/usage | head                                                                      # usage_*.jsonl
```

You build your pipeline in **this repo**. A dbt scaffold is provided in `transform/`.

---

## 2. About EndeavorIQ

EndeavorIQ is a fictional B2B SaaS **workflow-automation platform** (think Zapier/Retool).
Customers ("accounts") pay a per-seat subscription on a plan tier (Starter / Team / Business /
Enterprise) plus metered **usage** (API calls, automations, storage) above plan limits.
Accounts sign up, trial, convert, expand/contract seats, change plans, and sometimes churn.

You don't need deep domain knowledge — but the business shape (recurring + usage revenue,
seat changes, churn) is what your models will express.

---

## 3. The source landscape

Three source systems, each a different modality. Treat them as **external systems you don't
control** — they are messy in realistic ways.

### A. App OLTP database — **Postgres** (`app-db`, schema `app`)
The product's operational DB and the subscription source of truth.
`accounts`, `users`, `plans`, `subscriptions`, `subscription_changes`, plus `users_cdc`.

- Every table has `created_at` / `updated_at`; `accounts`/`subscriptions` have `deleted_at`
  (soft delete, set on churn).
- **`users` rows are *hard*-deleted** (GDPR erasure) — they vanish from the table with no flag.
  A **`users_cdc` audit log** (`seq, op ∈ {I,U,D}, …, changed_at`) records every insert/update/
  delete, so deletes are recoverable there.
- `plans` is **effective-dated** (`effective_from`/`effective_to`/`is_current`) — prices change
  over time, so "the plan as of date X" matters.
- `subscriptions` mutate in place (seat/plan/status changes); `subscription_changes` logs them.

### B. Billing provider API — **REST** (`billing-api`, `http://localhost:8080`)
A Stripe-flavored, read-only API over the billing system. The money source of truth.
`/v1/customers`, `/v1/invoices`, `/v1/charges`, `/v1/refunds`.

- **Cursor pagination**: `?limit=100&starting_after=<id>`, response has `has_more`. You must
  page through everything.
- Timestamps are **epoch seconds**; amounts are **integer cents**.
- Invoice **line items are nested** under `lines.data` (subscription + usage lines).
- `customer.metadata.account_id` links a billing customer to an app account — but it is
  **sometimes absent** (you may be able to recover the link another way; some can't be linked
  at all).

### C. Usage metering events — **JSONL files** (`drop/usage/usage_YYYY-MM-DD.jsonl`)
One file per day, one JSON object per line (`event_id, account_id, usage_date, metric, quantity`).
The metering source of truth.

- Lines may be **out of order**; a field may **appear partway through history** (schema drift);
  a file may contain **duplicate lines**; some days' files may be **missing**; and after a
  `tick` (below) a file for an **earlier date can arrive late**.

---

## 4. The CDC mechanic — how to exercise incremental loads

`make seed` lays down history. To simulate the source systems *changing*, advance time:

```bash
make tick DAYS=14    # advances simulated time: new signups, seat/plan changes, churns,
                     # user hard-deletes, new invoices, new usage files (+ a late-arriving one)
```

The intended loop:
1. `make seed`, run your pipeline (full load).
2. `make tick DAYS=14`.
3. Run your pipeline again — it should **capture the changes** (including deletes) **without
   duplicating or losing rows**, and a run with **no tick in between should change nothing**
   (idempotent).

The simulator is deterministic, but **don't hardcode anything to specific values** — we grade
against a freshly generated world.

---

## 5. Your tasks

### Core path (what we expect everyone to attempt)

**1. Extract-Load.** Land **raw** copies of all three sources into a DuckDB database. Make it
**incremental and idempotent** per the loop in §4. EL approach is your choice.

**2. Source-faithfulness manifest.** Fill in `manifest.yml` (template provided): for each source
entity, which of your tables is meant to be 1:1 faithful to it, its key, and how your columns map
to the canonical columns we check. This drives our objective scoring (§6) — so we can compare your
tables to the source regardless of how you named things.

**3. Staging models (dbt).** In `transform/`, build `stg_*` models: cleaned, cast, **deduped**,
1:1 with the sources. Add `schema.yml` tests (unique / not_null / relationships / accepted_values)
and at least one or two custom data-quality checks targeting issues you find.

**4. The mart that matters — `fct_revenue`.** Recognized revenue per **account × month**:
**recurring** (subscription seats × plan price, with **annual plans normalized to monthly**)
**plus** **usage** overage, with MRR-movement classification (new / expansion / contraction /
churn). Getting this right needs point-in-time subscription/plan state and the cross-source joins.

**5. Writeup (`WRITEUP.md`).** Brief: your EL/CDC strategy, the data-quality issues you found,
key modeling decisions, and **what you'd do with more time / how you'd productionize** (orchestration,
scheduling, alerting, the parts you stubbed). Include **how you used AI** (2–3 sentences): where it
helped, and at least one place where it was wrong, incomplete, or you overrode it. We're not testing
*whether* you used AI — we're testing whether you were driving. Write it in your own voice.

### Stretch (optional, no penalty if skipped)
- A **reconciliation** model: billed (invoices) vs collected (charges) vs expected (subscriptions).
- Reconstruct the user hard-deletes **without** `users_cdc` (DIY change detection).
- WAL / logical-replication-style CDC off Postgres.
- `dim_accounts` (SCD2 from `subscription_changes` / `updated_at`), more tests, docs.

**Constraints:** **DuckDB + dbt are required** for the warehouse + transforms. EL tooling is your
choice. A single entrypoint (`make run`, a script — whatever) must run EL → dbt end-to-end.

---

## 6. How we evaluate

Two tiers:

- **Objective (automated).** Using your `manifest.yml`, we compare your source-faithful staging
  tables against the true current source state — checking **key sets** (so deletes are handled),
  **row counts**, and **values** on the declared columns. We run this after a full load, after a
  `tick` (tests incremental + deletes), and on a no-tick re-run (tests idempotency). This scores
  EL correctness without caring about your modeling choices.
- **Judgment.** Your `fct_revenue` logic, dbt structure/naming, test coverage, and writeup.

| | Weight | What we look at |
|---|---|---|
| EL correctness & robustness | 30% | incremental, idempotent, deletes captured, pagination/files handled |
| dbt modeling & `fct_revenue` | 25% | layering, grain, SCD/point-in-time, annual normalization |
| Data-quality rigor | 20% | tests + which issues you caught & documented |
| Reproducibility | 10% | one clean command, no manual fixups |
| Writeup & judgment | 15% | priorities, honest gaps, productionization plan |

We do **not** grade on whether you finished everything, or on EL tool choice.

---

## 7. Submission

Submit this repo (zipped, or a private Git link) including:
- your **EL code**, your **`transform/`** dbt project, your filled-in **`manifest.yml`**, and **`WRITEUP.md`**;
- a note in `README.md` of anything we need to run it.

**Do not include** `node_modules/`, the DuckDB file, `drop/`, or other generated data.

Email your submission to **nathan@endeavorlabs.co**.

We'll follow up with a conversation to walk through your submission together — your EL/CDC and
modeling choices, what you'd do differently, and anything you wanted to try but didn't have time
for. Expect to extend the pipeline or write a model live. Submitting work you can explain and own
is the whole point.

---

## Troubleshooting
- **`make seed` failed / counts look wrong** — `make reset && make seed`. If it persists, email us with the error.
- **A question is ambiguous** — make a reasonable choice, document it in `WRITEUP.md`, move on.
- **Something else is wrong** — email us. We'd rather help than have you stuck.

Good luck, and have fun with it.
