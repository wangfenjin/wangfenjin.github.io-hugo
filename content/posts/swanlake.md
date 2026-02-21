---
title: "SwanLake：一个基于 DuckDB + DuckLake 的 Arrow Flight SQL 数据湖服务"
date: "2026-02-21"
author: "Wang Fenjin"
tags: ["swanlake", "duckdb", "arrow-flight-sql", "datalake", "rust"]
keywords: ["swanlake", "duckdb", "flight sql", "datalake", "duckdb-rs"]
description: "SwanLake 是我在 duckdb-rs 之后的继续探索：把 DuckDB 的能力通过 Arrow Flight SQL 变成可部署、可观测的数据服务。"
---

2023 年我把 [duckdb-rs](https://github.com/duckdb/duckdb-rs) 交给 DuckDB 官方维护之后，心里一直有个没做完的题：

如果 DuckDB 在单机进程里已经足够强，那怎么把这份能力变成一个更容易接入、可部署、可观测的服务？

[SwanLake](https://github.com/swanlake-io/swanlake) 就是我给这个问题的答案。

它本质上是一个基于 Rust 的 Arrow Flight SQL Server，底层执行引擎是 DuckDB，同时围绕 DuckLake 做了数据湖场景的能力扩展。更准确地说，SwanLake 的核心组合是 DuckDB + DuckLake + Arrow Flight SQL。

<div style="text-align: center;">
  <img src="/img/swanlake-overview.jpeg" alt="SwanLake 项目示意图" style="width: 760px; max-width: 100%; height: auto;" />
</div>

## 为什么是 SwanLake

我最早做 duckdb-rs 的时候，目标是把 DuckDB 更自然地带到 Rust 生态里。这个目标后来基本实现了，但新的问题也很明确：

1. 很多团队不是 Rust 单一语言栈，客户端接入方式需要统一。
2. 业务里常见的不只是“本地查询”，而是对象存储 + 元数据 + 多服务协作。
3. 线上系统需要可观测性，不能只靠日志排障。

所以 SwanLake 从一开始就不是“再封一层 API”，而是想做一个可以真正放进生产系统的分析服务入口。

## 系统架构

SwanLake 可以按 5 层来理解：

### 1) 接入层：Arrow Flight SQL（gRPC）

服务对外暴露 Flight SQL 接口，查询和更新请求都从这里进入。这个协议的核心价值是跨语言和高吞吐，仓库里的 Rust/Go/Python 示例就是围绕这层展开的。

### 2) 会话层：Session Registry

`swanlake-core` 里有连接级会话管理：

1. 按 `peer_addr` 或 `peer_ip` 生成/复用会话 ID。
2. prepared statement、事务、临时对象都跟随会话。
3. 通过最大会话数和空闲超时做资源保护。

### 3) 执行层：DuckDB

执行层没有重新造轮子，而是把 DuckDB 封装成服务可用的执行引擎。每个会话持有独立连接，启动时会加载 `ducklake/httpfs/aws/postgres` 扩展，并支持通过 `SWANLAKE_DUCKLAKE_INIT_SQL` 注入初始化 SQL。

### 4) 数据湖层：DuckLake

[DuckLake](https://ducklake.select/) 是这个系统最关键的一层。没有 DuckLake，DuckDB 更多是本地分析引擎；有了 DuckLake，元数据和对象存储路径就能以统一方式组织起来，SwanLake 才能把“DuckDB 做数据湖”变成可部署的服务方案。

### 5) 运维层：Metrics + Status + 配置

运行时指标（延迟、慢查询、错误）、状态页（`/` + `status.json`）和环境变量配置（`SWANLAKE_*`）共同组成了运维面。这个层的目标是让系统上线后可观测、可调优、可回滚。

## 可观测性

SwanLake 内置了状态页（默认 `:4215`）和 `status.json`，会展示：

1. 当前会话数、空闲时长等会话状态。
2. Query/Update 延迟统计（平均、P95、P99）。
3. 慢查询和最近错误。

<div style="text-align: center;">
  <img src="/img/swanlake-status-page.png" alt="SwanLake 状态页" style="width: 760px; max-width: 100%; height: auto;" />
</div>

我做这个页面不是为了“好看”，而是因为这正是我自己排查问题时最想第一时间看到的数据。

## 当前 benchmark，我怎么看

仓库里的 `BENCHMARK.md`（2026-02-21）有一组 TPCH（SF=0.1）结果：`postgres_local_file` 相比 `postgres_s3` 在这轮测试里更快。

<div style="max-width: 760px; margin: 0 auto; overflow-x: auto;">
  <table>
    <thead>
      <tr>
        <th>指标（SF=0.1）</th>
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

这个结果在预期之内：对象存储链路会引入更高的不确定性。

另外这里有一个非常重要的实践建议：如果后端是 S3 或其他远程对象存储，建议默认启用 [`cache_httpfs`](https://duckdb.org/community_extensions/extensions/cache_httpfs)，否则延迟（尤其 tail latency）会非常不稳定。

这个策略我已经放进项目基准流程里了，具体可以直接看 [`.github/workflows/performance.yml`](https://github.com/swanlake-io/swanlake/blob/main/.github/workflows/performance.yml)：

1. `postgres_s3` 默认 `BENCHBASE_ENABLE_CACHE_HTTPFS=true`。
2. `postgres_local_file` 默认 `BENCHBASE_ENABLE_CACHE_HTTPFS=false`。
3. 也可以通过 workflow input 显式覆盖该参数。

但我不想把它简单总结成“本地一定更好”。更准确的结论是：

1. 你需要根据 workload 做分层（热数据、本地缓存、远端对象存储）。
2. 你需要反复跑 benchmark 看方差，而不是拿一次结果定架构。
3. 你需要把指标做成持续可见的数据，而不是一次性报告。

## 从 duckdb-rs 到 SwanLake

如果说 duckdb-rs 解决的是“如何让开发者在 Rust 里优雅地用 DuckDB”，那 SwanLake 解决的是另一个问题：

如何把 DuckDB 变成一个团队可共享的、可部署的、可运维的数据服务。

这两个项目对我来说是一条连续的技术路线，而不是两个孤立项目。

## 后面还会做什么

SwanLake 还在持续迭代，我接下来会继续重点做几件事：

1. 继续补齐生产场景下的稳定性与压测数据。
2. 优化对象存储场景的性能和可预测性。
3. 让客户端和服务端的使用体验更统一，降低接入门槛。

如果你之前用过 duckdb-rs，我也很欢迎你来试试 SwanLake，提 issue、提 PR、或者直接分享你遇到的问题。

## 参考

* [DuckDB 官网](https://duckdb.org/)
* [DuckLake 官网](https://ducklake.select/)
* [Arrow Flight SQL 官网](https://arrow.apache.org/docs/format/FlightSql.html)
* [SwanLake 官网](https://swanlake.io)
* [SwanLake 代码](https://github.com/swanlake-io/swanlake)
