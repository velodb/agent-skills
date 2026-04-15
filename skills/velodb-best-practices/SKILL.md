---
name: velodb-best-practices
description: >
  VeloDB/Apache Doris table design and cluster sizing best practices.
  MUST USE when writing, reviewing, or optimizing Doris CREATE TABLE statements,
  partition/bucket strategies, data models, or cluster configurations.
  Also use when user provides a VeloDB connection string or asks to get started.
  Also use when user mentions velo CLI, velocli, or wants to connect to VeloDB/Doris.
license: Apache-2.0
metadata:
  author: VeloDB
  version: "3.1.0"
---

# VeloDB Best Practices

> Problem-first table design intelligence for Apache Doris.
> 37 rules, 7 use case templates, 4 sizing guides.
> All details in `references/` directory and compiled `AGENTS.md`.

---

## 1 ▸ Problem-First Routing

### I need to build…

| Problem | Template(s) | Key Rules |
|---------|-------------|-----------|
| Real-time log/event analytics | `usecase-log-event` | DUPLICATE, RANGE partition, dynamic TTL, ZSTD |
| CDC / MySQL sync to Doris | `usecase-cdc-sync` | UNIQUE MoW, sequence_col, HASH bucket |
| Dashboard with pre-aggregated metrics | `usecase-dashboard-metrics` | AGGREGATE, BITMAP_UNION, sync MV |
| User-facing API with low-latency point queries | `usecase-point-query` | UNIQUE MoW, store_row_column, BloomFilter |
| Star schema with JOIN-heavy analytics | `usecase-star-schema-join` | Colocation, same bucket key/count |
| Small dimension / lookup table | `usecase-dimension-lookup` | DUPLICATE, RANDOM bucket, 3 buckets |
| Observability (logs + traces + metrics) | `usecase-observability` | 3 tables: DUP logs, DUP traces, AGG metrics |
| Vehicle/fleet tracking | `usecase-log-event` + `usecase-point-query` | Time-series + point-query hybrid |
| E-commerce order analytics | `usecase-star-schema-join` + `usecase-dashboard-metrics` | Star schema + AGG rollups |
| Full-text search / content search | `schema-index-text-search` | Inverted index, MATCH, BM25 |
| User behavior / funnel analysis | `schema-types-bitmap-count-distinct` | BITMAP_UNION, bitmap_intersect |
| Semi-structured JSON data | `schema-types-variant-json` | VARIANT type, schema_template |

### My query is slow because…

| Symptom | Check These Rules | Quick Fix |
|---------|------------------|-----------|
| Full table scan on WHERE clause | `schema-keys-selectivity-first` | Move filtered column to sort key position 1 |
| JOINs are slow / shuffle | `usecase-star-schema-join` | Small dims (<1GB): broadcast + runtime filter. Large: colocation |
| COUNT DISTINCT is slow | `schema-types-bitmap-count-distinct` | Switch to BITMAP_UNION aggregation |
| LIKE '%keyword%' is slow | `schema-index-ngram-for-like` | Add NGram BloomFilter index |
| Point query latency too high | `usecase-point-query` | Enable store_row_column + Prepared Statement |
| Storage growing too fast | `schema-partition-auto-on-demand` + `schema-props-compression` | AUTO PARTITION + ZSTD compression + scheduled DROP PARTITION |
| Sync MV not being used | `schema-mv-sync-rollup` | Use raw columns (not date_trunc) in MV GROUP BY; unique aliases |
| Async MV rewrite fails | `schema-mv-async-join` + `schema-mv-async-limits` | Check State/RefreshState; query MV directly if predicate fails |
| Data skew / hot tablets | `schema-bucket-composite-for-skew` | Composite bucket key or RANDOM |
| Import fails / data version error | `schema-mv-async-limits` | Check concurrent MV refresh limit (max 3) |
| VARCHAR in key kills perf | `schema-keys-fixed-length-types` | Move VARCHAR after fixed-length types |
| Writes slow on UNIQUE table | `schema-model-prefer-mow` | Ensure MoW is enabled (not MoR) |

---

## 2 ▸ Pre-Flight Checklist (Before Any CREATE TABLE)

Run through this checklist in order. Each step references the relevant rule:

- [ ] **Data model** — UNIQUE (updates?) vs DUPLICATE (append?) vs AGGREGATE (pre-agg only?) → `schema-model-choose-for-workload`
- [ ] **Partition strategy** — Time-series? AUTO PARTITION preferred. Small table? Skip. Do NOT combine AUTO with dynamic_partition. → `schema-partition-*`
- [ ] **Bucket key + count** — HASH on JOIN key. Calculate explicit count: `daily_GB / target_tablet_GB`. Avoid BUCKETS AUTO. → `schema-bucket-*`
- [ ] **Sort key order** — High-selectivity first, fixed-length before VARCHAR → `schema-keys-*`
- [ ] **Data types** — Native types, not STRING. DECIMAL not FLOAT. → `schema-types-*`
- [ ] **Indexes** — BloomFilter for equality, Inverted for text, NGram for LIKE → `schema-index-*`
- [ ] **Properties** — MoW enabled? Compression? Cloud mode replication_num=1? → `schema-props-*`

---

## 3 ▸ Connection & VeloCLI

### Detect VeloCLI

Before running any queries, check for the `velo` CLI tool:

1. Check `VELO_CLI_PATH` env var — if set, use that binary path
2. Otherwise run `which velo` — if found, use it from PATH
3. If neither: fall back to `mysql` client (see `references/start-*.md`)

### When VeloCLI is available, prefer it for all operations:

| Task | VeloCLI Command |
|------|-----------------|
| Run SQL | `velo sql "SELECT ..."` |
| DDL inspection | `velo sql "SHOW CREATE TABLE db.t"` |
| Table/tablet health | `velo tablet db.t` (overview) or `velo tablet db.t --detail` |
| Profile a slow query | `velo sql "SELECT ..." --profile` → captures query_id |
| Get query profile | `velo profile get <qid>` or `--full` for complete diagnosis |
| Compare fast vs slow | `velo profile diff <slow_qid> <fast_qid>` |
| Performance trend | `velo profile history <sql_pattern> --days 7` |
| Test connection | `velo auth status` |
| Switch environment | `velo use <name>` |

### Quick-start guides

- `references/start-cloud.md` — VeloDB Cloud
- `references/start-self-hosted.md` — Self-hosted / BYOC / on-prem

---

## 4 ▸ Cluster Sizing

Sizing guides are in:
- `references/sizing-fe.md` — FE node sizing
- `references/sizing-be-integrated.md` — BE sizing (integrated storage)
- `references/sizing-be-cloud.md` — BE sizing (cloud / storage-compute)
- `references/sizing-storage-formula.md` — Storage calculation formula

---

## 5 ▸ Rule Index by Category

### Data Model — CRITICAL (4 rules)
- `schema-model-choose-for-workload` — DUP vs UNIQUE vs AGG decision tree
- `schema-model-prefer-mow` — Always MoW for UNIQUE tables
- `schema-model-avoid-agg-for-updates` — AGG cannot UPDATE/DELETE
- `schema-model-sequence-col-for-cdc` — Sequence column for out-of-order CDC

### Partition Strategy — CRITICAL (4 rules)
- `schema-partition-range-for-timeseries` — RANGE for time-series
- `schema-partition-dynamic-ttl` — Dynamic partition for automated TTL
- `schema-partition-auto-on-demand` — AUTO for sporadic data
- `schema-partition-skip-for-small` — Skip partitioning under 1 GB

### Bucket Strategy — CRITICAL (5 rules)
- `schema-bucket-hash-vs-random` — HASH for pruning, RANDOM for DUP only
- `schema-bucket-high-cardinality-key` — Choose high-cardinality column
- `schema-bucket-composite-for-skew` — Composite key to fix data skew
- `schema-bucket-target-size` — Target 1-10 GB per tablet
- `schema-bucket-cloud-mandatory-hash` — Cloud MoW requires HASH

### Sort Key — CRITICAL (5 rules)
- `schema-keys-selectivity-first` — High selectivity first
- `schema-keys-fixed-length-types` — Fixed-length before VARCHAR
- `schema-keys-prefix-index-limits` — 36 bytes max, VARCHAR terminates it
- `schema-keys-cluster-key-for-mow` — Cluster key for UNIQUE tables
- `schema-keys-avoid-float` — No FLOAT/DOUBLE in sort key

### Data Types — HIGH (5 rules)
- `schema-types-native-vs-string` — Native types, not STRING
- `schema-types-zonemap-limitations` — JSON/ARRAY disable ZoneMap
- `schema-types-variant-json` — VARIANT for semi-structured JSON
- `schema-types-bitmap-count-distinct` — BITMAP_UNION for exact count-distinct
- `schema-types-doris-specifics` — DATETIME precision, VARCHAR vs STRING

### Indexes — HIGH (7 rules)
- `schema-index-bloomfilter` — BloomFilter for equality
- `schema-index-inverted` — Inverted for text/range
- `schema-index-ngram-for-like` — NGram for LIKE %pattern%
- `schema-index-bitmap` — Bitmap for medium cardinality
- `schema-index-vector` — HNSW/IVF for ANN search
- `schema-index-text-search` — Full-text MATCH + BM25

### Query Acceleration — HIGH (3 rules)
- `schema-mv-sync-rollup` — Sync MV for single-table aggregation
- `schema-mv-async-join` — Async MV for multi-table JOIN
- `schema-mv-async-limits` — Operational limits (50M rows, 3 concurrent)

### Table Properties — HIGH/MEDIUM (2 rules)
- `schema-props-cloud-forced` — Cloud mode forced properties
- `schema-props-compression` — LZ4 vs ZSTD compression

### Caching — MEDIUM (2 rules)
- `schema-cache-file-cache` — File cache for cloud mode
- `schema-cache-query-partition` — Query and partition cache
