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
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lib/httpserver/httpserver.go                           │  │
│  │  • ReadHeaderTimeout: 5s (读取请求头超时)               │  │
│  │  • IdleConnTimeout: 1min (空闲连接超时)                 │  │
│  │  • ConnTimeout: 2min (连接生命周期)                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 2: 并发控制                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lib/writeconcurrencylimiter/concurrencylimiter.go      │  │
│  │  • -maxConcurrentInserts: 2×CPU (最大并发写入数)        │  │
│  │  • -insert.maxQueueDuration: 1min (排队超时)            │  │
│  │  触发行为: 超时返回 HTTP 503                            │  │
│  │  监控指标: vm_concurrent_insert_limit_timeout_total     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 3: 请求大小限制                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lib/protoparser/*/stream/streamparser.go               │  │
│  │  • -maxInsertRequestSize: 32MB (Prometheus)             │  │
│  │  • -influx.maxInsertRequestSize: 32MB (InfluxDB)        │  │
│  │  • -datadog.maxInsertRequestSize: 64MB (DataDog)        │  │
│  │  触发行为: 拒绝请求，返回错误                           │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 4: 数据验证                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vminsert/main.go                                   │  │
│  │  • -maxLabelsPerTimeseries: 30 (每 series 最大标签数)   │  │
│  │  • -maxLabelValueLen: 16KB (标签值最大长度)             │  │
│  │  触发行为: 超限标签被丢弃/截断                          │  │
│  │  监控指标: vm_metrics_with_dropped_labels_total         │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 5: 基数限制 (Cardinality Limiting) ⭐关键配置          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lib/storage/storage.go                                 │  │
│  │  • -storage.maxHourlySeries: 0 (每小时最大新 series)    │  │
│  │  • -storage.maxDailySeries: 0 (每天最大新 series)       │  │
│  │  触发行为: 超限 series 的数据被丢弃                     │  │
│  │  监控指标: vm_hourly_series_limit_rows_dropped_total    │  │
│  │           vm_daily_series_limit_rows_dropped_total      │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 6: 存储层保护                                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  lib/storage/storage.go & partition.go                  │  │
│  │  • -storage.minFreeDiskSpaceBytes: 10MB (最小磁盘空间)  │  │
│  │    → 低于阈值进入只读模式                               │  │
│  │  • maxInmemoryParts: 60 (内存 parts 上限)               │  │
│  │    → 达到上限时写入暂停，等待 merge                     │  │
│  │  • maxRawRowsPerShard: 8MB (分片缓冲上限)               │  │
│  │    → 超限自动 flush                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
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
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/main.go                                   │  │
│  │  • -search.maxConcurrentRequests: 基于CPU计算           │  │
│  │  • -search.maxQueueDuration: 10s (排队超时)             │  │
│  │  触发行为: 超时返回 HTTP 429 (Too Many Requests)        │  │
│  │  监控指标: vm_concurrent_select_limit_timeout_total     │  │
│  │           vm_concurrent_select_current (当前并发数)     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 2: 请求验证                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/prometheus/prometheus.go                  │  │
│  │  app/vmselect/searchutils/searchutils.go                │  │
│  │  • -search.maxQueryLen: 16KB (查询语句最大长度)         │  │
│  │  • -search.maxQueryDuration: 30s (查询超时)             │  │
│  │  触发行为: 拒绝请求 / 超时中断                          │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 3: 搜索范围控制                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/prometheus/prometheus.go                  │  │
│  │  app/vmselect/netstorage/netstorage.go                  │  │
│  │  • -search.maxUniqueTimeseries: 300000 (最大 series)    │  │
│  │  • -search.maxTagKeys: 100000 (最大 tag keys)           │  │
│  │  • -search.maxTagValues: 100000 (最大 tag values)       │  │
│  │  触发行为: 返回 "too many time series" 错误             │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 4: 数据量控制                                          │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/netstorage/netstorage.go                  │  │
│  │  • -search.maxSamplesPerQuery: 1B (每查询最大采样点)    │  │
│  │  • -search.maxSamplesPerSeries: 30M (每series最大采样点)│  │
│  │  触发行为: 返回 "too many samples" 错误                 │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 5: 执行资源控制                                        │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/netstorage/netstorage.go                  │  │
│  │  app/vmselect/promql/eval.go                            │  │
│  │  • -search.maxWorkersPerQuery: CPU核数 (并行度)         │  │
│  │  • -search.maxMemoryPerQuery: 0 (单查询内存限制)        │  │
│  │  • Deadline 持续检查 (贯穿整个执行过程)                 │  │
│  │  触发行为: 限制并行度 / 拒绝请求 / 超时中断             │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  Layer 6: 结果控制                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app/vmselect/prometheus/prometheus.go                  │  │
│  │  • -search.maxPointsPerTimeseries: 30000 (返回点数限制) │  │
│  │  触发行为: 数据截断                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│  监控与日志                                                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  • -search.logSlowQueryDuration: 5s (慢查询日志阈值)    │  │
│  │  • -search.logQueryMemoryUsage: 0 (内存使用日志阈值)    │  │
│  │  监控指标: vm_slow_queries_total                        │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                           返回查询结果
```

---

## VictoriaMetrics 限流相关参数配置

### 写入端参数

```bash
# ==================== vminsert / 单机版 ====================

# --- 并发控制 ---
-maxConcurrentInserts=16              # 最大并发写入数，默认 2×CPU
-insert.maxQueueDuration=1m           # 写入排队超时，默认 1 分钟

# --- 请求大小限制 ---
-maxInsertRequestSize=33554432        # 单次写入请求最大 32MB (Prometheus 协议)
-influx.maxInsertRequestSize=33554432 # InfluxDB 协议请求大小限制
-datadog.maxInsertRequestSize=67108864 # DataDog 协议请求大小限制

# --- 标签限制 ---
-maxLabelsPerTimeseries=30            # 每个 series 最大标签数
-maxLabelValueLen=16384               # 标签值最大长度 16KB


# ==================== vmstorage / 单机版 ====================

# --- 基数限制 (防止高基数导致 OOM，强烈建议配置) ---
-storage.maxHourlySeries=500000       # 每小时最大新增 series 数
-storage.maxDailySeries=2000000       # 每天最大新增 series 数

# --- 磁盘空间保护 ---
-storage.minFreeDiskSpaceBytes=10485760  # 最小剩余磁盘空间，默认 10MB
                                         # 低于此值进入只读模式

# --- 内存限制 ---
-memory.allowedPercent=60             # 允许使用的内存百分比，默认 60%

# --- Merge 控制 (影响 CPU 和 IO) ---
-bigMergeConcurrency=1                # 大 merge 并发数
-smallMergeConcurrency=2              # 小 merge 并发数
```

### 查询端参数

```bash
# ==================== vmselect / 单机版 ====================

# --- 并发控制 ---
-search.maxConcurrentRequests=16      # 最大并发查询数，默认基于 CPU 计算
-search.maxQueueDuration=10s          # 查询排队超时，默认 10 秒

# --- 超时控制 ---
-search.maxQueryDuration=60s          # 单次查询最大执行时间，默认 30 秒

# --- 查询长度 ---
-search.maxQueryLen=16384             # 查询语句最大长度，默认 16KB

# --- 数据量限制 ---
-search.maxUniqueTimeseries=300000    # 单次查询最大 series 数
-search.maxSamplesPerQuery=1000000000 # 单次查询最大采样点数 (1B)
-search.maxSamplesPerSeries=30000000  # 单个 series 最大采样点数 (30M)
-search.maxPointsPerTimeseries=30000  # 返回结果每 series 最大点数

# --- 资源限制 ---
-search.maxWorkersPerQuery=0          # 每查询最大 worker 数，0 表示使用 CPU 核数
-search.maxMemoryPerQuery=0           # 单次查询最大内存，0 表示不限制

# --- 慢查询日志 ---
-search.logSlowQueryDuration=5s       # 超过此时间的查询记录到日志
-search.logQueryMemoryUsage=0         # 超过此内存的查询记录到日志
```

### 推荐生产配置模板

**64GB 内存机器配置示例：**

```bash
# vmstorage 启动参数
-storageDataPath=/data/victoriametrics
-retentionPeriod=30d
-memory.allowedPercent=60
-storage.maxHourlySeries=500000
-storage.maxDailySeries=2000000
-storage.minFreeDiskSpaceBytes=10737418240  # 10GB
-bigMergeConcurrency=1
-smallMergeConcurrency=2
-loggerLevel=WARN

# vminsert 启动参数
-maxConcurrentInserts=16
-maxLabelsPerTimeseries=30
-maxLabelValueLen=4096
-insert.maxQueueDuration=1m

# vmselect 启动参数
-search.maxConcurrentRequests=8
-search.maxQueryDuration=60s
-search.maxUniqueTimeseries=300000
-search.maxSamplesPerQuery=500000000
-search.maxMemoryPerQuery=4294967296  # 4GB
-search.logSlowQueryDuration=10s
```

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
    │       └─→ 高基数问题 → 启用基数限制 + 清理高基数指标
    │
    ├─→ vm_active_merges 持续很高?
    │       └─→ 写入压力过大 → 降低写入并发 + 上游限流
    │
    └─→ vm_concurrent_select_current 很高?
            └─→ 查询压力过大 → 限制查询并发 + 优化大查询
```

#### 处理方案

**高基数问题处理：**
```bash
# 1. 启用基数限制（需重启）
-storage.maxHourlySeries=300000
-storage.maxDailySeries=1000000

# 2. 配置 relabel 丢弃高基数指标
# /etc/vminsert/relabel.yml
- source_labels: [__name__]
  regex: "problem_metric_.*"
  action: drop
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
| `regexp` | 正则匹配 | 优化查询 |
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
    │       ├─→ 有大查询 → 限制查询 + 通知用户优化
    │       └─→ 慢查询增多 → 配置 -search.maxQueryDuration
    │
    ├─→ vmstorage CPU 高?
    │       ├─→ merge 活跃数高 → 降低 merge 并发
    │       └─→ searchTSIDs 热点 → 高基数问题
    │
    └─→ vminsert CPU 高?
            ├─→ 写入 QPS 突增 → 上游限流
            └─→ relabel 热点 → 简化 relabel 规则
```

#### 处理方案

**大查询处理：**
```bash
# 限制查询
-search.maxQueryDuration=30s
-search.maxSamplesPerQuery=100000000
-search.maxUniqueTimeseries=100000
```

**Merge 压力处理：**
```bash
# HDD 磁盘建议
-bigMergeConcurrency=1
-smallMergeConcurrency=2

# SSD 磁盘可适当提高
-bigMergeConcurrency=2
-smallMergeConcurrency=4
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
    │       ├─→ vm_active_merges{type="storage/big"} > 0
    │       │       └─→ 大 Merge 进行中，等待完成或降低并发
    │       ├─→ 写入 QPS 突增
    │       │       └─→ 上游限流
    │       └─→ indexdb merge 很高
    │               └─→ 高基数问题
    │
    ├─→ 读 IO 为主?
    │       ├─→ 有大查询
    │       │       └─→ 限制查询并发 + 优化查询
    │       └─→ 缓存未命中率高
    │               └─→ 增加缓存 / 等待预热
    │
    └─→ 正在备份/snapshot?
            └─→ 错峰执行备份
```

#### 处理方案

**大 Merge 处理：**
```bash
# 降低 merge 并发（会导致 parts 积压）
-bigMergeConcurrency=1
-smallMergeConcurrency=1

# 或等待 merge 完成（通常是最佳选择）
watch -n 5 'curl -s http://localhost:8482/metrics | grep vm_active_merges'
```

**缓存未命中处理：**
```bash
# 增加内存允许百分比（更多内存给缓存）
-memory.allowedPercent=70

# 等待缓存预热（重启后通常需要 10-30 分钟）
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
