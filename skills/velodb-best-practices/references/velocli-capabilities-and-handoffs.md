---
title: VeloCLI Capabilities and Manual Handoffs
tags: [velocli, workflow, handoff, cli, automation]
---
## VeloCLI Capabilities and Manual Handoffs

Use this map when driving an end-to-end workflow with `velo`
(benchmarks, demos, ad-hoc analysis, tuning loops). It tells you what
the CLI covers, where it doesn't, and how to step out of the way
cleanly so the user can finish the missing piece.

### What `velo` can do

| Capability                  | Command(s)                                                                       |
|-----------------------------|----------------------------------------------------------------------------------|
| Configure environment       | `velo auth add / list / status / remove`                                         |
| Warehouse: discover and use | `velo cloud warehouse ls / get / use`                                            |
| Cluster lifecycle           | `velo cloud cluster ls / get / create / use / resize / start / stop`             |
| Run SQL (DDL + DML)         | `velo sql "..."` or `velo sql -f file.sql`                                       |
| Force JSON for scripting    | `velo --format json ...` (the flag is on the root, before the subcommand)        |
| Profile by query id         | `velo profile list`, `velo profile get <qid>`, `velo profile diff <slow> <fast>` |
| Profile by SQL pattern      | `velo profile history <pattern>`                                                 |
| Tablet inspection           | `velo tablet ...`                                                                |

When in doubt, run `velo --help` and the relevant `velo <subcommand>
--help` before guessing.

### What `velo` CANNOT do — stop and hand off to the user

For each item below: **stop the workflow, tell the user exactly what
to do, give them the verbatim command(s), and wait for confirmation.**
Do not script around these from inside your turn.

#### A. Warehouse creation

There is no `velo cloud warehouse create`. Say:

> I can't create a warehouse from the CLI. Please create one in the
> SelectDB Cloud console (https://cloud.selectdb.com), then run
> `velo cloud warehouse ls` and paste the result — I'll continue.

Then wait.

#### B. Bulk data loading

There is no `velo load`. When you need to load any non-trivial amount
of data, **stop and present the user with the options below**, then
ask which path they want. Don't just pick one — the right answer
depends on dataset location and cluster network policy.

1. **S3 / HTTP TVF via `velo sql`** — single
   `INSERT INTO ... SELECT * FROM S3(...)` statement. Works only when
   (a) the dataset URL is publicly reachable and (b) the cluster has
   outbound network to that host. Fast path when it works; the agent
   can run it directly.

2. **Stream-load via curl, user-driven** — the agent is not in a good
   position to drive this for large files (long-running, opaque
   progress, possible auth + network surprises). Give the user the
   exact template and wait:

   ```bash
   curl --location-trusted \
       -u <user>:<password> \
       -T <local_file> \
       -H "Expect: 100-continue" \
       -H "columns:<comma_separated_columns_in_file_order>" \
       http://<host>:<stream_load_port>/api/<db>/<table>/_stream_load
   ```

   Get `<host>` and `<stream_load_port>` from
   `velo cloud cluster get <name>` → `endpoint` section. Get
   credentials with `velo cloud cluster password` or from the console.

3. **External orchestration** — Airflow / Spark / Doris connector /
   Doris Streamloader. Out of scope for the CLI; if the user is
   already in such a pipeline, tell them which file to load into
   which table and let them drive.

**To hand off cleanly:**
- State what needs to load (table, source URI, row count if known).
- Recommend the path most likely to work given cluster region and
  source location.
- Print the exact command with placeholders filled in where possible.
- Tell the user: "run this, then tell me when it's done — I'll
  verify with `SELECT COUNT(*)`."
- Wait.

When the user comes back, verify with
`velo sql "SELECT COUNT(*) FROM <table>"` before continuing.

#### C. Cluster billing model / region switch

No CLI exposure. Console-only.

### Quirks that bite

- `velo sql -f FILE` is **single-statement**. Don't put `USE ...;` or
  chained `DROP ... ; CREATE ...` in a file — only the first runs.
  Use fully-qualified `db.table` and split files.
- Every `velo sql` call opens a fresh connection. No session state
  carries across calls (no `SET`, no `USE`, no temp tables).
- Server-side file cache is not flushed between runs. When
  benchmarking, take the min of N tries and label it
  "warm-cache best of N".
- `--no-cache` only disables VeloCLI's local result cache, not the
  server's file cache.
- `exec_time_ms` in the JSON response is client-measured (includes
  RTT). For high-RTT setups (SOCKS5, cross-region), prefer the
  server-side execution time from `velo profile get <qid>`.
- `ANALYZE TABLE <t> WITH SYNC` — if the cluster rejects `WITH SYNC`,
  fall back to `ANALYZE TABLE <t>`.

### How this combines with the design rules

The `schema-*`, `usecase-*`, and `sizing-*` references in this skill
tell you **what** to design and how to diagnose. This file tells you
**what tools** to reach for and **when to step out of the way**. Both
apply.
