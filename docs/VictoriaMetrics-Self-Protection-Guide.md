# VictoriaMetrics 是如何实现自我限流保护的

## 背景

夜莺的很多用户时序存储都是用的 VictoriaMetrics，偶尔会遇到 VictoriaMetrics OOM、CPU 飙高、IO 打满等问题的反馈。本文通过阅读 VictoriaMetrics 源码，系统性地介绍 VictoriaMetrics 的限流保护机制，帮助大家更好地理解和运维 VictoriaMetrics。

VictoriaMetrics 在写入和查询两条路径上都实现了多层次的限流保护机制，从网络层、请求层、数据层到存储层，形成了完整的自我保护体系。理解这些机制，可以帮助我们：

1. 合理配置参数，避免资源耗尽
2. 快速定位问题根因
3. 制定有效的应急处理方案

---

## VictoriaMetrics 架构

在深入限流机制之前，我们先了解 VictoriaMetrics 的整体架构。

### 部署模式

VictoriaMetrics 支持两种部署模式：

**单机模式 (victoria-metrics)**
```
┌─────────────────────────────────────────────────────┐
│         victoria-metrics (单一进程)                 │
├─────────────────────────────────────────────────────┤
│  ┌───────────────┐  ┌───────────────┐  ┌─────────┐ │
│  │   vminsert    │  │   vmstorage   │  │vmselect │ │
│  │   (写入层)    │──│   (存储层)    │──│(查询层) │ │
│  └───────────────┘  └───────────────┘  └─────────┘ │
└─────────────────────────────────────────────────────┘
              默认端口: 8428
```

**集群模式**
```
┌─────────────────────────────────────────────────────────────────┐
│  数据源 (Prometheus, vmagent, Telegraf 等)                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      vminsert (写入层)                          │
│  • 接收多种协议数据 (Prometheus, InfluxDB, OpenTSDB 等)         │
│  • 通过一致性哈希路由到 vmstorage                               │
│  • 默认端口: 8480                                               │
└────────────────────┬────────────────────┬───────────────────────┘
                     │                    │
                     ▼                    ▼
        ┌────────────────────┐  ┌────────────────────┐
        │   vmstorage #1     │  │   vmstorage #2     │  ...
        │  • 存储时序数据    │  │  • 存储时序数据    │
        │  • 维护索引        │  │  • 维护索引        │
        │  • 默认端口: 8482  │  │  • 默认端口: 8482  │
        └────────────────────┘  └────────────────────┘
                     │                    │
        ┌────────────┴────────────────────┴────────────┐
        │               vmselect (查询层)              │
        │  • 分发查询请求到所有 vmstorage              │
        │  • 合并聚合查询结果                          │
        │  • 默认端口: 8481                            │
        └──────────────────────┬───────────────────────┘
                               │
                               ▼
                     查询客户端 (Grafana 等)
```

### 核心组件职责

| 组件 | 职责 | 关键限流点 |
|------|------|------------|
| **vminsert** | 数据写入路由 | 并发控制、请求大小、标签验证 |
| **vmstorage** | 数据存储引擎 | 基数限制、磁盘保护、内存背压 |
| **vmselect** | 查询执行引擎 | 并发控制、超时控制、数据量限制 |

### 存储结构

```
/data/victoriametrics/
├── data/                    # 时序数据
│   └── {YYYY_MM}/          # 按月分区
│       ├── small/          # 小 parts (频繁写入)
│       └── big/            # 大 parts (merge 后)
├── indexdb/                 # 索引数据 (metric name + labels → TSID)
│   ├── {current_db}/       # 当前索引库
│   └── {prev_db}/          # 上一个索引库 (轮换)
├── cache/                   # 缓存数据
├── snapshots/              # 快照
└── metadata/               # 元数据
```

**关键概念：**
- **TSID**: Time Series ID，每个唯一的 metric + labels 组合对应一个 TSID
- **Part**: 数据存储单元，由多个 Block 组成
- **Merge**: 后台将多个小 Part 合并为大 Part 的过程
- **IndexDB**: 存储 metric name + labels 到 TSID 的映射索引

---

## VictoriaMetrics 限流机制总览图

### 写入路径限流层次图

```
                           数据写入请求
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 1: 网络层保护                                          │
│  • 读取请求头超时                                             │
│  • 空闲连接超时                                               │
│  • 连接生命周期限制                                           │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 2: 并发控制                                            │
│  • 最大并发写入数限制                                         │
│  • 排队超时控制                                               │
│  • 触发行为: 超时返回 HTTP 503                                │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 3: 请求大小限制                                        │
│  • 单次请求最大字节数                                         │
│  • 支持多种协议独立配置                                       │
│  • 触发行为: 拒绝请求                                         │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 4: 数据验证                                            │
│  • 每 series 最大标签数                                       │
│  • 标签值最大长度                                             │
│  • 触发行为: 超限标签被丢弃/截断                              │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 5: 基数限制 (Cardinality Limiting) ⭐关键               │
│  • 每小时最大新 series 数                                     │
│  • 每天最大新 series 数                                       │
│  • 触发行为: 超限 series 的数据被丢弃                         │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 6: 存储层保护                                          │
│  • 最小磁盘空间保护 → 低于阈值进入只读模式                    │
│  • 内存 parts 上限 → 达到上限时写入暂停                       │
│  • 分片缓冲上限 → 超限自动 flush                              │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                            数据落盘
```

### 查询路径限流层次图

```
                           查询请求
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 1: 并发控制                                            │
│  • 最大并发查询数限制                                         │
│  • 排队超时控制                                               │
│  • 触发行为: 超时返回 HTTP 429                                │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 2: 请求验证                                            │
│  • 查询语句最大长度                                           │
│  • 单次查询最大执行时间                                       │
│  • 触发行为: 拒绝请求 / 超时中断                              │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 3: 搜索范围控制                                        │
│  • 单次查询最大 series 数                                     │
│  • 最大 tag keys / tag values 数                              │
│  • 触发行为: 返回 "too many time series" 错误                 │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 4: 数据量控制                                          │
│  • 每查询最大采样点数                                         │
│  • 每 series 最大采样点数                                     │
│  • 触发行为: 返回 "too many samples" 错误                     │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 5: 执行资源控制                                        │
│  • 每查询最大 worker 数                                       │
│  • 单查询内存限制                                             │
│  • 执行过程持续检查 Deadline                                  │
│  • 触发行为: 限制并行度 / 拒绝请求 / 超时中断                 │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 6: 结果控制                                            │
│  • 返回结果每 series 最大点数                                 │
│  • 触发行为: 数据截断                                         │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  监控与日志                                                   │
│  • 慢查询日志记录                                             │
│  • 高内存查询日志记录                                         │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                           返回查询结果
```

---

## VictoriaMetrics 限流相关参数配置

以下参数按照限流层次顺序介绍，适用于集群版本部署。

### vminsert 参数（写入层）

#### Layer 2: 并发控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-maxConcurrentInserts` | 2×CPU | 最大并发写入请求数。超过此数量的请求会排队等待 |
| `-insert.maxQueueDuration` | 1m | 写入请求排队等待的最大时间。超时返回 HTTP 503 |

#### Layer 3: 请求大小限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-maxInsertRequestSize` | 32MB | Prometheus remote write 协议单次请求最大字节数 |
| `-influx.maxInsertRequestSize` | 32MB | InfluxDB 协议单次请求最大字节数 |
| `-datadog.maxInsertRequestSize` | 64MB | DataDog 协议单次请求最大字节数 |
| `-opentsdbhttp.maxInsertRequestSize` | 32MB | OpenTSDB HTTP 协议单次请求最大字节数 |

#### Layer 4: 数据验证

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-maxLabelsPerTimeseries` | 30 | 每个 series 允许的最大标签数。超出的标签会被丢弃 |
| `-maxLabelValueLen` | 16KB | 单个标签值的最大长度。超出部分会被截断 |

### vmstorage 参数（存储层）

#### Layer 5: 基数限制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-storage.maxHourlySeries` | 0 (不限制) | 每小时允许新增的最大 series 数。**强烈建议配置此参数防止高基数导致 OOM** |
| `-storage.maxDailySeries` | 0 (不限制) | 每天允许新增的最大 series 数 |

#### Layer 6: 存储层保护

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-storage.minFreeDiskSpaceBytes` | 10MB | 最小剩余磁盘空间。低于此值存储进入只读模式，拒绝所有写入 |
| `-memory.allowedPercent` | 60% | 允许使用的系统内存百分比。用于控制缓存大小 |
| `-bigMergeConcurrency` | CPU/2 | 大数据块 merge 的并发数。影响 CPU 和 IO |
| `-smallMergeConcurrency` | CPU/2 | 小数据块 merge 的并发数。影响 CPU 和 IO |
| `-retentionPeriod` | 1 month | 数据保留时间 |

### vmselect 参数（查询层）

#### Layer 1: 并发控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxConcurrentRequests` | 2×CPU (最大16) | 最大并发查询请求数。超过此数量的请求会排队等待 |
| `-search.maxQueueDuration` | 10s | 查询请求排队等待的最大时间。超时返回 HTTP 429 |

#### Layer 2: 请求验证

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxQueryLen` | 16KB | 查询语句的最大长度 |
| `-search.maxQueryDuration` | 30s | 单次查询的最大执行时间。超时会中断查询 |

#### Layer 3: 搜索范围控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxUniqueTimeseries` | 300000 | 单次查询允许匹配的最大 series 数 |
| `-search.maxTagKeys` | 100000 | /api/v1/labels 接口返回的最大 tag keys 数 |
| `-search.maxTagValues` | 100000 | /api/v1/label/.../values 接口返回的最大 tag values 数 |

#### Layer 4: 数据量控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxSamplesPerQuery` | 1000000000 | 单次查询允许扫描的最大采样点数 |
| `-search.maxSamplesPerSeries` | 30000000 | 单个 series 允许返回的最大采样点数 |

#### Layer 5: 执行资源控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxWorkersPerQuery` | CPU 核数 | 单次查询使用的最大并行 worker 数 |
| `-search.maxMemoryPerQuery` | 0 (不限制) | 单次查询允许使用的最大内存 |

#### Layer 6: 结果控制

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.maxPointsPerTimeseries` | 30000 | 返回结果中每个 series 的最大数据点数 |

#### 监控与日志

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-search.logSlowQueryDuration` | 5s | 执行时间超过此值的查询会记录到日志 |
| `-search.logQueryMemoryUsage` | 0 (不记录) | 内存使用超过此值的查询会记录到日志 |

---

## VictoriaMetrics 遇到容量问题 SOP

### VictoriaMetrics 内存暴涨 SOP

#### 快速止血（1-5 分钟）

**1. 确认服务状态**
```bash
# 查看进程内存占用
ps aux | grep -E 'vmstorage|vmselect|vminsert|victoria-metrics' | awk '{print $2, $4, $6/1024"MB", $11}'

# 查看系统内存
free -h

# 查看 OOM 日志
dmesg | grep -i "out of memory" | tail -20
```

**2. 紧急限流**
```bash
# 方案 A: 临时阻断写入
iptables -I INPUT -p tcp --dport 8480 -j DROP   # vminsert 端口
iptables -I INPUT -p tcp --dport 8428 -j DROP   # 单机版端口

# 方案 B: 重启并限制内存
systemctl stop vmstorage
# 添加 -memory.allowedPercent=40 后重启
systemctl start vmstorage
```

#### 问题定位

**1. 查看关键指标**
```bash
curl -s http://localhost:8482/metrics | grep -E "^(process_resident_memory|vm_rows|vm_new_timeseries|vm_hourly_series|vm_active_merges|vm_parts)"
```

| 指标 | 含义 | 异常判断 |
|------|------|----------|
| `process_resident_memory_bytes` | 进程内存 | 持续上涨 |
| `vm_new_timeseries_created_total` | 新建 series | 突增说明高基数 |
| `vm_active_merges{type="indexdb"}` | IndexDB 合并 | 持续高说明索引膨胀 |

**2. 检查基数**
```bash
# 查看总 series 数
curl -s "http://localhost:8428/api/v1/status/tsdb" | jq '.data.totalSeries'

# 访问 cardinality 页面
# http://localhost:8428/vmui/#/cardinality
```

**3. 检查数据目录**
```bash
du -sh /data/victoriametrics/*
# 重点关注 indexdb 大小，高基数会导致 indexdb 膨胀
```

#### 根因分析决策树

```
内存暴涨
    │
    ├─→ indexdb 过大 (>10GB)?
    │       │
    │       └─→ 是: 高基数问题
    │           ├─→ 检查 vm_new_timeseries_created_total 增速
    │           ├─→ 检查 cardinality 页面找出高基数指标
    │           └─→ 跳转【场景A: 高基数处理】
    │
    ├─→ vm_active_merges 持续很高?
    │       │
    │       └─→ 是: 写入压力过大，merge 跟不上
    │           ├─→ 检查 vm_rows_added_to_storage_total 增速
    │           ├─→ 检查 vm_parts 数量是否持续增长
    │           └─→ 跳转【场景B: 写入压力处理】
    │
    ├─→ vm_concurrent_select_current 很高?
    │       │
    │       └─→ 是: 查询压力过大
    │           ├─→ 检查 active_queries 找出大查询
    │           ├─→ 检查 vm_rows_read_per_query 值
    │           └─→ 跳转【场景C: 查询压力处理】
    │
    └─→ process_resident_memory 突然跳涨?
            │
            └─→ 是: 可能是单个大查询或大批量写入
                ├─→ 检查 vmselect 日志找出大查询
                ├─→ 检查 vminsert 日志找出大批量写入
                └─→ 跳转对应场景处理
```

#### 处理方案

**场景A: 高基数处理**
```bash
# 1. 启用基数限制（需重启）
-storage.maxHourlySeries=300000
-storage.maxDailySeries=1000000

# 2. 配置 relabel 丢弃高基数指标（vminsert 配置）
# /etc/vminsert/relabel.yml
- source_labels: [__name__]
  regex: "problem_metric_.*"
  action: drop

# 3. 通知业务方优化指标（减少高基数标签）
```

**场景B: 写入压力处理**
```bash
# 1. 降低写入并发
-maxConcurrentInserts=8

# 2. 上游限流（vmagent/prometheus 端降低抓取频率）

# 3. 扩容 vmstorage 节点
```

**场景C: 查询压力处理**
```bash
# 1. 限制查询并发
-search.maxConcurrentRequests=4

# 2. 限制单次查询数据量
-search.maxSamplesPerQuery=100000000

# 3. 通知用户优化查询
```

#### 预防告警规则

```yaml
- alert: VMHighMemoryUsage
  expr: process_resident_memory_bytes / on() machine_memory_bytes > 0.8
  for: 5m
  annotations:
    summary: "VictoriaMetrics 内存使用率超过 80%"

- alert: VMHighSeriesCreationRate
  expr: rate(vm_new_timeseries_created_total[5m]) > 1000
  for: 10m
  annotations:
    summary: "新 series 创建速率过高，可能存在高基数问题"

- alert: VMCardinalityLimitReached
  expr: rate(vm_hourly_series_limit_rows_dropped_total[5m]) > 0
  for: 1m
  annotations:
    summary: "基数限制已触发，正在丢弃数据"
```

---

### VictoriaMetrics CPU 消耗突然变高 SOP

#### 快速止血（1-5 分钟）

**1. 确认 CPU 状态**
```bash
# 查看进程 CPU 占用
top -p $(pgrep -d',' -f 'vmstorage|vmselect|vminsert|victoria-metrics')

# 查看各组件 CPU
ps aux | grep -E 'vmstorage|vmselect|vminsert' | awk '{print $11, "CPU:"$3"%"}'
```

**2. 快速判断原因**

| 组件 | CPU 高的常见原因 |
|------|------------------|
| **vmselect** | 复杂查询、大范围查询、正则匹配 |
| **vmstorage** | merge 操作、索引查找、解压缩 |
| **vminsert** | 高并发写入、relabel 处理 |

**3. 紧急限流**
```bash
# vmselect CPU 高
-search.maxConcurrentRequests=2

# vmstorage CPU 高
-smallMergeConcurrency=1
-bigMergeConcurrency=1
```

#### 问题定位

**1. 使用 pprof 抓取 CPU Profile**
```bash
# 抓取 30 秒的 CPU profile
curl "http://localhost:8482/debug/pprof/profile?seconds=30" > cpu.prof
go tool pprof -top cpu.prof | head -30
```

**常见 CPU 热点函数：**

| 函数关键字 | 含义 | 处理方向 |
|-----------|------|----------|
| `mergeParts` | 数据合并 | 降低 merge 并发 |
| `searchTSIDs` | 索引搜索 | 高基数问题 |
| `regexp` | 正则匹配 | 优化查询，避免复杂正则 |
| `decompress` | 解压缩 | 大量数据读取 |
| `relabel` | 重标签 | 简化 relabel 规则 |

**2. 查看活跃查询**
```bash
curl -s "http://localhost:8481/api/v1/status/active_queries" | jq .
curl -s "http://localhost:8481/api/v1/status/top_queries" | jq '.topByDuration[:10]'
```

**3. 查看 merge 状态**
```bash
curl -s http://localhost:8482/metrics | grep vm_active_merges
```

#### 根因分析决策树

```
CPU 突然变高
    │
    ├─→ vmselect CPU 高?
    │       │
    │       ├─→ active_queries 中有大查询?
    │       │       │
    │       │       └─→ 是: 大查询导致
    │       │           ├─→ 检查查询时间范围是否过大
    │       │           ├─→ 检查是否使用了复杂正则
    │       │           ├─→ 检查返回的 series 数量
    │       │           └─→ 跳转【场景A: 大查询处理】
    │       │
    │       └─→ vm_slow_queries_total 增长快?
    │               │
    │               └─→ 是: 慢查询过多
    │                   ├─→ 检查 top_queries 找出问题查询
    │                   └─→ 跳转【场景B: 慢查询优化】
    │
    ├─→ vmstorage CPU 高?
    │       │
    │       ├─→ vm_active_merges 很高?
    │       │       │
    │       │       └─→ 是: merge 压力大
    │       │           ├─→ 检查 vm_parts 数量
    │       │           ├─→ 检查写入速率
    │       │           └─→ 跳转【场景C: Merge 压力处理】
    │       │
    │       └─→ pprof 显示 searchTSIDs 热点?
    │               │
    │               └─→ 是: 索引搜索压力大
    │                   ├─→ 通常是高基数问题
    │                   └─→ 参考内存 SOP 的高基数处理
    │
    └─→ vminsert CPU 高?
            │
            ├─→ 写入 QPS 突增?
            │       │
            │       └─→ 是: 写入压力大
            │           ├─→ 检查上游数据源
            │           └─→ 跳转【场景D: 写入压力处理】
            │
            └─→ pprof 显示 relabel 热点?
                    │
                    └─→ 是: relabel 规则复杂
                        ├─→ 检查 relabel 配置
                        ├─→ 简化正则表达式
                        └─→ 减少 relabel 规则数量
```

#### 处理方案

**场景A: 大查询处理**
```bash
# 1. 限制查询时间和数据量
-search.maxQueryDuration=30s
-search.maxSamplesPerQuery=100000000
-search.maxUniqueTimeseries=100000

# 2. 通知用户优化查询
# - 缩小时间范围
# - 添加更精确的 label 过滤
# - 避免使用 {__name__=~".+"} 这类全量查询
```

**场景B: 慢查询优化**
```bash
# 1. 启用慢查询日志
-search.logSlowQueryDuration=5s

# 2. 分析 top_queries 找出问题查询模式
# 常见问题：
# - 时间范围过大（如 30d）
# - 正则过于复杂
# - 缺少 label 过滤
```

**场景C: Merge 压力处理**
```bash
# HDD 磁盘建议
-bigMergeConcurrency=1
-smallMergeConcurrency=2

# SSD 磁盘可适当提高
-bigMergeConcurrency=2
-smallMergeConcurrency=4
```

**场景D: 写入压力处理**
```bash
# 1. 降低写入并发
-maxConcurrentInserts=8

# 2. 上游限流
# - 增加 scrape_interval
# - 减少抓取目标
# - 使用 vmagent stream aggregation 预聚合
```

#### 预防告警规则

```yaml
- alert: VMHighCPUUsage
  expr: rate(process_cpu_seconds_total{job=~".*victoria.*"}[5m]) > 0.8
  for: 10m
  annotations:
    summary: "VictoriaMetrics CPU 使用率超过 80%"

- alert: VMSlowQueriesIncreasing
  expr: rate(vm_slow_queries_total[5m]) > 1
  for: 10m
  annotations:
    summary: "慢查询数量持续增长"

- alert: VMMergeBacklog
  expr: vm_parts{type="storage/small"} > 50
  for: 15m
  annotations:
    summary: "Storage parts 积压，merge 可能跟不上"
```

---

### VictoriaMetrics IO 消耗突然变高 SOP

#### 快速止血（1-5 分钟）

**1. 确认 IO 状态**
```bash
# 查看磁盘 IO
iostat -xz 1 5

# 查看进程 IO
iotop -oP

# 关键指标
# %util > 80% 需关注
# await > 10ms (SSD) 或 > 20ms (HDD) 需关注
```

**2. 判断 IO 类型**
```bash
iostat -x 1 3 | awk '/sd|nvme/{print $1, "r/s:"$4, "w/s:"$5, "util:"$NF"%"}'
```

| IO 类型 | 常见原因 |
|---------|----------|
| **写 IO 高** | 大量写入、merge 操作、索引更新 |
| **读 IO 高** | 大范围查询、缓存未命中 |
| **读写都高** | merge 操作（读旧写新） |

**3. 紧急限流**
```bash
# 写 IO 高
-maxConcurrentInserts=4
-bigMergeConcurrency=1
-smallMergeConcurrency=1

# 读 IO 高
-search.maxConcurrentRequests=2
-search.maxSamplesPerQuery=50000000
```

#### 问题定位

**1. 查看 VictoriaMetrics IO 指标**
```bash
curl -s http://localhost:8482/metrics | grep -E "^(vm_active_merges|vm_parts|vm_cache_misses|vm_rows)"
```

| 指标 | 含义 | 异常判断 |
|------|------|----------|
| `vm_active_merges` | 活跃 merge 数 | 多个大 merge 同时进行 |
| `vm_parts` | parts 数量 | 过多说明 merge 积压 |
| `vm_cache_misses_total` | 缓存未命中 | 快速增长需要读盘 |
| `vm_assisted_merges_total` | 辅助 merge | 增长说明 merge 严重跟不上 |

**2. 查看 merge 状态**
```bash
curl -s http://localhost:8482/metrics | grep vm_active_merges
# vm_active_merges{type="storage/big"} > 0 说明大 merge 进行中
```

**3. 查看缓存命中率**
```bash
curl -s http://localhost:8482/metrics | grep -E "vm_cache_(requests|misses)_total"
# 命中率 = 1 - (misses / requests)，低于 90% 需关注
```

#### 根因分析决策树

```
IO 突然变高
    │
    ├─→ 写 IO 为主?
    │       │
    │       ├─→ vm_active_merges{type="storage/big"} > 0?
    │       │       │
    │       │       └─→ 是: 大 Merge 正在进行
    │       │           ├─→ 这是正常的后台操作
    │       │           ├─→ 检查 merge 预计完成时间
    │       │           └─→ 跳转【场景A: 大 Merge 处理】
    │       │
    │       ├─→ vm_rows_added_to_storage_total 突增?
    │       │       │
    │       │       └─→ 是: 写入量突增
    │       │           ├─→ 检查上游数据源
    │       │           └─→ 跳转【场景B: 写入压力处理】
    │       │
    │       └─→ vm_active_merges{type="indexdb"} 很高?
    │               │
    │               └─→ 是: IndexDB merge 压力大
    │                   ├─→ 通常是高基数导致
    │                   └─→ 参考内存 SOP 的高基数处理
    │
    ├─→ 读 IO 为主?
    │       │
    │       ├─→ active_queries 中有大查询?
    │       │       │
    │       │       └─→ 是: 查询读取大量数据
    │       │           ├─→ 检查查询时间范围
    │       │           ├─→ 检查返回的 series 数量
    │       │           └─→ 跳转【场景C: 大查询处理】
    │       │
    │       └─→ vm_cache_misses_total 增长快?
    │               │
    │               └─→ 是: 缓存未命中率高
    │                   ├─→ 可能刚重启，缓存未预热
    │                   ├─→ 可能缓存配置过小
    │                   ├─→ 可能查询了大量冷数据
    │                   └─→ 跳转【场景D: 缓存未命中处理】
    │
    └─→ 读写都高?
            │
            ├─→ 正在进行 snapshot/backup?
            │       │
            │       └─→ 是: 备份操作导致
            │           └─→ 跳转【场景E: 备份 IO 处理】
            │
            └─→ 多个大 merge 同时进行?
                    │
                    └─→ 是: Merge 操作读旧写新
                        └─→ 跳转【场景A: 大 Merge 处理】
```

#### 处理方案

**场景A: 大 Merge 处理**
```bash
# 1. 降低 merge 并发（会导致 parts 积压）
-bigMergeConcurrency=1
-smallMergeConcurrency=1

# 2. 或等待 merge 完成（通常是最佳选择）
# 大 merge 是正常的后台操作，完成后 IO 会恢复
watch -n 5 'curl -s http://localhost:8482/metrics | grep vm_active_merges'

# 3. 根据磁盘类型长期优化
# HDD: -bigMergeConcurrency=1 -smallMergeConcurrency=2
# SSD: -bigMergeConcurrency=2 -smallMergeConcurrency=4
```

**场景B: 写入压力处理**
```bash
# 1. 降低写入并发
-maxConcurrentInserts=4

# 2. 上游限流
# - 增加 scrape_interval
# - 减少抓取目标

# 3. 扩容 vmstorage 节点
```

**场景C: 大查询处理**
```bash
# 1. 限制查询并发和数据量
-search.maxConcurrentRequests=2
-search.maxSamplesPerQuery=50000000

# 2. 通知用户优化查询
```

**场景D: 缓存未命中处理**
```bash
# 1. 如果刚重启，等待缓存预热（通常需要 10-30 分钟）

# 2. 增加内存允许百分比（更多内存给缓存）
-memory.allowedPercent=70

# 3. 检查是否有大量冷数据查询
```

**场景E: 备份 IO 处理**
```bash
# 1. 错峰执行备份（避开业务高峰）

# 2. 限制备份并发
vmbackup -concurrency=1

# 3. 使用 snapshot 方式（IO 影响更小）
curl http://localhost:8482/snapshot/create
```

#### 预防告警规则

```yaml
- alert: VMDiskUtilizationHigh
  expr: rate(node_disk_io_time_seconds_total[5m]) > 0.8
  for: 10m
  annotations:
    summary: "磁盘 IO 利用率超过 80%"

- alert: VMBigMergeInProgress
  expr: vm_active_merges{type="storage/big"} > 0
  for: 30m
  annotations:
    summary: "大 Merge 正在进行，IO 可能较高"

- alert: VMPartsBacklog
  expr: vm_parts{type="storage/small"} > 50
  for: 15m
  annotations:
    summary: "Parts 积压，merge 可能跟不上写入"

- alert: VMAssistedMergesHigh
  expr: rate(vm_assisted_merges_total[5m]) > 0
  for: 5m
  annotations:
    summary: "触发辅助 Merge，写入速度超过 Merge 速度"
```

---

## 日常巡检 Checklist

| 检查项 | 命令/指标 | 正常范围 |
|--------|----------|----------|
| 内存使用率 | `process_resident_memory_bytes` | <70% |
| CPU 使用率 | `rate(process_cpu_seconds_total[5m])` | <70% |
| 磁盘 IO | `iostat -x` %util | <70% |
| Series 总数 | `/api/v1/status/tsdb` | 根据业务定 |
| 新 Series 速率 | `rate(vm_new_timeseries_created_total[1h])` | <10000/h |
| IndexDB 大小 | `du -sh indexdb/` | <data 目录 2 倍 |
| 活跃查询数 | `vm_concurrent_select_current` | <capacity 的 80% |
| Parts 数量 | `vm_parts{type="storage/small"}` | <30 |
| 慢查询数 | `rate(vm_slow_queries_total[1h])` | 增长平稳 |

---

## 总结

VictoriaMetrics 实现了完善的多层限流保护机制：

1. **写入端保护链**：并发控制 → 请求大小 → 标签验证 → 基数限制 → 磁盘保护 → 内存背压
2. **查询端保护链**：并发控制 → 超时控制 → 范围限制 → 数据量限制 → 资源限制 → 结果控制
3. **渐进式降级**：排队 → 丢弃 → 只读，尽可能保证服务可用性

**关键配置建议：**
- 务必配置 `-storage.maxHourlySeries` 和 `-storage.maxDailySeries` 防止高基数
- 根据机器配置调整 `-memory.allowedPercent`
- 配置告警规则，提前发现问题
- 定期检查 cardinality 页面，及时发现高基数指标

合理配置这些限流参数并建立监控告警，可以有效避免 OOM、CPU 飙高、IO 打满等问题，保障 VictoriaMetrics 的稳定运行。
