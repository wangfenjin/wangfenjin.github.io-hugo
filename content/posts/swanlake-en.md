---
title: "SwanLake: An Arrow Flight SQL Datalake Service Built on DuckDB + DuckLake"
date: "2026-02-21"
author: "Wang Fenjin"
tags: ["swanlake", "duckdb", "arrow-flight-sql", "datalake", "rust"]
keywords: ["swanlake", "duckdb", "flight sql", "datalake", "duckdb-rs"]
description: "SwanLake is my continuation after duckdb-rs: turning DuckDB into a deployable, observable Arrow Flight SQL service for real workloads."
---

After handing [duckdb-rs](https://github.com/duckdb/duckdb-rs) over to the DuckDB team in 2023, one question kept coming back to me:

If DuckDB is already great in-process, how do we turn that power into a service that is easier to integrate, deploy, and operate?

[SwanLake](https://github.com/swanlake-io/swanlake) is my answer to that question.

It is a Rust-based Arrow Flight SQL server, powered by DuckDB, with DuckLake-oriented extensions for datalake scenarios. In practice, SwanLake is built around a three-part combination: DuckDB + DuckLake + Flight SQL.

<div style="text-align: center;">
  <img src="/img/swanlake-overview.jpeg" alt="SwanLake project overview" style="width: 760px; max-width: 100%; height: auto;" />
</div>

## Why I started SwanLake

With duckdb-rs, the primary goal was clear: make DuckDB feel natural in Rust. That part worked well, but new constraints became obvious:

1. Most teams are not single-language; they need one service interface across stacks.
2. Real workloads involve object storage, metadata services, and multiple cooperating systems.
3. Production systems need observability, not just logs.

So SwanLake was never “just another wrapper”. I wanted a practical analytics service entrypoint.

## Architecture

You can read SwanLake as a five-layer system:

### 1) Access Layer: Arrow Flight SQL (gRPC)

All query/update traffic enters through Flight SQL. This gives us a protocol that is efficient and language-neutral; the Rust/Go/Python examples in the repo validate this layer directly.

### 2) Session Layer: Session Registry

`swanlake-core` manages connection-scoped sessions:

1. session IDs are created/reused from `peer_addr` or `peer_ip`,
2. prepared statements, transactions, and temp objects remain session-affine,
3. max sessions + idle timeout protect server resources.

### 3) Execution Layer: DuckDB

I did not build a new engine. SwanLake wraps DuckDB for service use: each session has an isolated connection, startup preloads `ducklake/httpfs/aws/postgres` extensions, and `SWANLAKE_DUCKLAKE_INIT_SQL` can inject bootstrap SQL.

### 4) Datalake Layer: DuckLake

[DuckLake](https://ducklake.select/) is the key piece. Without it, DuckDB is mainly an excellent local analytical engine. With DuckLake, metadata and object-storage paths can be organized consistently, which makes DuckDB-based datalake services practical.

### 5) Operations Layer: Metrics + Status + Config

Runtime metrics (latency/slow query/errors), status endpoints (`/` + `status.json`), and env-based configuration (`SWANLAKE_*`) form the operational surface. This layer is what makes the system observable and manageable in production.

## Observability was a first-class requirement

SwanLake has a built-in status page (default `:4215`) plus `status.json` for machine consumption. It exposes:

1. session counts and idle indicators,
2. query/update latency stats (avg, p95, p99),
3. slow query and recent error history.

<div style="text-align: center;">
  <img src="/img/swanlake-status-page.png" alt="SwanLake status page" style="width: 760px; max-width: 100%; height: auto;" />
</div>

I added this because these are exactly the signals I want when debugging production behavior.

## How I read the current benchmark data

`BENCHMARK.md` (CI artifact dated 2026-02-21) includes TPCH results at SF=0.1 where `postgres_local_file` outperforms `postgres_s3` in that run.

<div style="max-width: 760px; margin: 0 auto; overflow-x: auto;">
  <table>
    <thead>
      <tr>
        <th>Metric (SF=0.1)</th>
        <th style="text-align: right;">postgres_local_file</th>
        <th style="text-align: right;">postgres_s3</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Throughput (req/s)</td>
        <td style="text-align: right;">10.428</td>
        <td style="text-align: right;">4.867</td>
      </tr>
      <tr>
        <td>Avg latency (ms)</td>
        <td style="text-align: right;">382.751</td>
        <td style="text-align: right;">818.041</td>
      </tr>
      <tr>
        <td>p95 latency (ms)</td>
        <td style="text-align: right;">829.236</td>
        <td style="text-align: right;">1904.023</td>
      </tr>
      <tr>
        <td>p99 latency (ms)</td>
        <td style="text-align: right;">1116.002</td>
        <td style="text-align: right;">2661.619</td>
      </tr>
    </tbody>
  </table>
</div>

This is expected directionally: object storage paths usually add more variability.

One practical point is critical here: when using S3 or other remote object storage, you should usually enable [`cache_httpfs`](https://duckdb.org/community_extensions/extensions/cache_httpfs), otherwise latency, especially tail latency, can become very unstable.

This is already reflected in the benchmark workflow configuration. See [`.github/workflows/performance.yml`](https://github.com/swanlake-io/swanlake/blob/main/.github/workflows/performance.yml):

1. `postgres_s3` defaults to `BENCHBASE_ENABLE_CACHE_HTTPFS=true`,
2. `postgres_local_file` defaults to `BENCHBASE_ENABLE_CACHE_HTTPFS=false`,
3. the workflow input can override this behavior.

But I do not think the takeaway is “local is always better”. A better takeaway is:

1. choose storage tiers based on workload shape,
2. run repeated benchmarks and track variance,
3. keep performance visibility continuous, not one-off.

## From duckdb-rs to SwanLake

For me, duckdb-rs and SwanLake are part of the same line of work.

duckdb-rs solved: how to use DuckDB elegantly inside Rust applications.

SwanLake solves: how to provide DuckDB as a shared, deployable, operable service for teams.

## What I will keep working on

SwanLake is still evolving. My near-term focus is:

1. more production-oriented reliability and load testing,
2. better performance predictability on object storage backends,
3. a more consistent developer experience across server and clients.

If you used duckdb-rs before, I would love you to try SwanLake and share feedback via issues or PRs.

## References

* [DuckDB official website](https://duckdb.org/)
* [DuckLake official website](https://ducklake.select/)
* [Arrow Flight SQL documentation](https://arrow.apache.org/docs/format/FlightSql.html)
* [SwanLake official website](https://swanlake.io)
* [SwanLake source code](https://github.com/swanlake-io/swanlake)
