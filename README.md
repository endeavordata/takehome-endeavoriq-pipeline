# EndeavorIQ — Data Engineer Take-Home (Pipeline Build)

**👉 Read [`ASSIGNMENT.md`](./ASSIGNMENT.md) first — that's the full brief.**

You'll build an extract-load + dbt pipeline over three simulated source systems (a Postgres
OLTP DB, a Stripe-flavored billing API, and a JSONL usage-event file drop), all running locally
via Docker Compose.

## Quick start
```bash
make up              # pull + start Postgres and the billing API
make seed            # generate ~18 months of history across the three sources
make tick DAYS=14    # advance simulated time (to exercise incremental loads); see ASSIGNMENT §4
make reset           # wipe the app schema
make down            # stop everything
```
> First `make up`/`make seed` pulls the billing-API and simulator images from the GitHub
> Container Registry (`ghcr.io/endeavordata/…`, public — no login needed). Just have Docker running.

## Repo layout
```
ASSIGNMENT.md        # the brief — start here
manifest.yml         # you fill this in (source-faithfulness declaration; see ASSIGNMENT §5.2)
transform/           # dbt scaffold — build your staging + marts here
docker-compose.yml   # the three source systems (Postgres + pulled billing-api & simulator images)
schema/ddl.sql       # the Postgres DDL, for reference
```
The simulator and billing API ship as **pre-built images** (a black box) — there's no source to
read or modify. They populate the three sources you load *from*.

Put your **EL code** wherever you like in this repo. Add a one-command entrypoint (`make run`,
a script — your call) that runs EL → dbt. Write your findings in `WRITEUP.md`.

## Notes
- The sources are at **`localhost:5432`** (Postgres: `endeavoriq`/`endeavoriq`) and
  **`localhost:8080`** (billing API). Usage files land in **`drop/usage/`**.
- You'll need **dbt-duckdb** in your own environment for `transform/` (`pip install dbt-duckdb`).
- Don't commit the DuckDB file or generated data (see `.gitignore`).
