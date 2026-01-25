# VictoriaMetrics 是如何实现自我限流保护的

## 背景

夜莺的很多用户时序存储都是用的 VictoriaMetrics，偶尔会遇到 VictoriaMetrics OOM 如何处理的反馈。本周通过阅读 VictoriaMetrics 代码，介绍下 VictoriaMetrics 是如何做限流保护的。

VictoriaMetrics 在写入和查询两条路径上都实现了多层次的限流保护机制，从网络层、请求层、数据层到存储层，形成了完整的自我保护体系。

---

## 太长不看版

### 写入端限流配置

```bash
# vminsert / vmstorage 启动参数

# 并发写入控制
-maxConcurrentInserts=16              # 最大并发写入数，默认 2×CPU
-insert.maxQueueDuration=1m           # 写入排队超时，默认 1 分钟

# 请求大小限制
-maxInsertRequestSize=33554432        # 单次写入请求最大 32MB

# 基数限制 (重要！防止高基数导致 OOM)
-storage.maxHourlySeries=500000       # 每小时最大新增 series 数
-storage.maxDailySeries=2000000       # 每天最大新增 series 数

# 标签限制
-maxLabelsPerTimeseries=30            # 每个 series 最大标签数
-maxLabelValueLen=16384               # 标签值最大长度 16KB

# 磁盘空间保护
-storage.minFreeDiskSpaceBytes=10485760  # 最小剩余磁盘空间 10MB，低于此值进入只读模式

# 内存限制
-memory.allowedPercent=60             # 允许使用的内存百分比
```

### 查询端限流配置

```bash
# vmselect 启动参数

# 并发查询控制
-search.maxConcurrentRequests=16      # 最大并发查询数，默认基于 CPU 计算
-search.maxQueueDuration=10s          # 查询排队超时

# 查询超时
-search.maxQueryDuration=30s          # 单次查询最大执行时间

# 数据量限制
-search.maxUniqueTimeseries=300000    # 单次查询最大 series 数
-search.maxSamplesPerQuery=1000000000 # 单次查询最大采样点数
-search.maxSamplesPerSeries=30000000  # 单个 series 最大采样点数
-search.maxPointsPerTimeseries=30000  # 返回结果每 series 最大点数

# 查询长度
-search.maxQueryLen=16384             # 查询语句最大长度 16KB

# 内存限制
-search.maxMemoryPerQuery=0           # 单次查询最大内存，0 表示不限制

# 慢查询日志
-search.logSlowQueryDuration=5s       # 超过此时间的查询记录到日志
```

---

## 限流相关代码阅读

### 一、写入路径限流机制

VictoriaMetrics 的写入路径从 HTTP 请求接收到数据最终落盘，经过了多层限流保护。

#### 1.1 写入流程概览

```
HTTP 请求 → 协议解析 → 数据验证 → 批量缓冲 → TSID 查找 → 分区写入 → 内存 Part → 落盘
```

#### 1.2 并发写入控制

**代码位置**: `lib/writeconcurrencylimiter/concurrencylimiter.go`

```go
// Line 18-24: 配置定义
var (
    maxConcurrentInserts = flag.Int("maxConcurrentInserts", 2*cgroup.AvailableCPUs(),
        "The maximum number of concurrent insert requests")
    maxQueueDuration = flag.Duration("insert.maxQueueDuration", time.Minute,
        "The maximum duration to wait in the queue when -maxConcurrentInserts concurrent insert requests are executed")
)

// Line 90-92: 使用带缓冲的 channel 实现并发控制
var concurrencyLimitCh = make(chan struct{}, *maxConcurrentInserts)

// Line 67-78: 并发检查逻辑
func incConcurrency() bool {
    select {
    case concurrencyLimitCh <- struct{}{}:
        // 成功获取槽位
        return true
    default:
    }
    // 槽位已满，等待或超时
    t := timerpool.Get(*maxQueueDuration)
    select {
    case concurrencyLimitCh <- struct{}{}:
        timerpool.Put(t)
        return true
    case <-t.C:
        timerpool.Put(t)
        // 超时，返回 HTTP 503
        return false
    }
}
```

**触发行为**: 超时返回 HTTP 503，指标 `vm_concurrent_insert_limit_timeout_total` 增加。

#### 1.3 请求大小限制

不同协议有各自的请求大小限制：

| 协议 | 配置项 | 默认值 | 代码位置 |
|------|--------|--------|----------|
| Prometheus Remote Write | `-maxInsertRequestSize` | 32MB | `lib/protoparser/promremotewrite/stream/streamparser.go:20` |
| OpenTSDB HTTP | `-opentsdbhttp.maxInsertRequestSize` | 32MB | `lib/protoparser/opentsdbhttp/stream/streamparser.go:23` |
| DataDog | `-datadog.maxInsertRequestSize` | 64MB | `lib/protoparser/datadogutils/datadogutils.go:14` |
| InfluxDB | `-influx.maxInsertRequestSize` | 32MB | `lib/protoparser/influx/stream/streamparser.go` |

**代码示例** (`lib/protoparser/promremotewrite/stream/streamparser.go`):
```go
// Line 20
var maxInsertRequestSize = flagutil.NewBytes("maxInsertRequestSize", 32*1024*1024,
    "The maximum size in bytes of a single Prometheus remote_write API request")

// Line 67-68: 检查解压后的大小
if len(bb.B) > maxInsertRequestSize.IntN() {
    return fmt.Errorf("too big unpacked request; mustn't exceed -maxInsertRequestSize=%d bytes",
        maxInsertRequestSize.N)
}
```

#### 1.4 标签验证限制

**代码位置**: `app/vminsert/main.go`

```go
// Line 68-69
var (
    maxLabelsPerTimeseries = flag.Int("maxLabelsPerTimeseries", 30,
        "The maximum number of labels accepted per time series")
    maxLabelValueLen = flag.Int("maxLabelValueLen", 16*1024,
        "The maximum length of label value")
)
```

**处理方式**:
- 超过 `maxLabelsPerTimeseries` 的标签被**丢弃**，指标 `vm_metrics_with_dropped_labels_total` 增加
- 超过 `maxLabelValueLen` 的标签值被**截断**，指标 `vm_too_long_label_values_total` 增加

#### 1.5 基数限制 (Cardinality Limiting)

这是防止高基数导致 OOM 的关键配置。

**代码位置**: `lib/storage/storage.go`

```go
// Line 54: 配置定义
var (
    maxHourlySeries = flag.Int("storage.maxHourlySeries", 0,
        "The maximum number of unique series that can be added during the last hour")
    maxDailySeries = flag.Int("storage.maxDailySeries", 0,
        "The maximum number of unique series that can be added during the last 24 hours")
)

// Line 198-202: 使用布隆过滤器初始化限制器
if *maxHourlySeries > 0 {
    s.hourlySeriesLimiter = bloomfilter.NewLimiter(*maxHourlySeries, time.Hour)
}
if *maxDailySeries > 0 {
    s.dailySeriesLimiter = bloomfilter.NewLimiter(*maxDailySeries, 24*time.Hour)
}

// Line 1956-1962: 写入时检查
if !s.registerSeriesCardinality(r.MetricNameRaw) {
    // 超过基数限制，丢弃数据
    atomic.AddUint64(&s.hourlySeriesLimitRowsDropped, 1)
    continue
}
```

**触发行为**: 超限数据被直接丢弃，相关指标:
- `vm_hourly_series_limit_rows_dropped_total`
- `vm_daily_series_limit_rows_dropped_total`

#### 1.6 磁盘空间保护

**代码位置**: `lib/storage/storage.go`

```go
// Line 61: 配置
var minFreeDiskSpaceBytes = flagutil.NewBytes("storage.minFreeDiskSpaceBytes", 10*1024*1024,
    "The minimum free disk space at -storageDataPath after which the storage switches to read-only mode")

// Line 655-694: 后台监控 goroutine
func startFreeDiskSpaceWatcher() {
    for {
        freeSpace := fs.MustGetFreeSpace(storagePath)
        if freeSpace < minFreeDiskSpaceBytes.N {
            // 切换到只读模式
            storage.SetReadOnly(true)
            logger.Warnf("switching to read-only mode; free disk space: %d bytes", freeSpace)
        }
        time.Sleep(time.Second)
    }
}

// Line 150-158 (app/vmstorage/main.go): 写入时检查
func AddRows(mrs []storage.MetricRow) error {
    if Storage.IsReadOnly() {
        return fmt.Errorf("storage is in read-only mode")  // 返回 HTTP 503
    }
    // ...
}
```

#### 1.7 内存背压控制

**代码位置**: `lib/storage/partition.go`

```go
// Line 39: In-Memory Parts 数量限制
const maxInmemoryParts = 60

// Line 74: Raw Row 缓冲区大小
const maxRawRowsPerShard = (8 << 20) / int(unsafe.Sizeof(rawRow{}))  // 8MB per shard
```

**机制说明**:
- 当 in-memory parts 数量达到 60 时，写入暂停，等待后台 merge
- 每个 shard 的 raw row 缓冲区限制为 8MB
- Insert Context 缓冲区达到 16MB 时自动 flush (`app/vminsert/common/insert_ctx.go:93`)

#### 1.8 时间戳验证

**代码位置**: `lib/storage/storage.go:1759-1778`

```go
// 检查时间戳是否在 retention 范围内
if mr.Timestamp < minTimestamp {
    atomic.AddUint64(&s.tooSmallTimestampRows, 1)
    continue  // 丢弃过旧数据
}
if mr.Timestamp > maxTimestamp {
    atomic.AddUint64(&s.tooBigTimestampRows, 1)
    continue  // 丢弃过于未来的数据
}
```

**相关指标**:
- `vm_rows_ignored_total{reason="small_timestamp"}`
- `vm_rows_ignored_total{reason="big_timestamp"}`

---

### 二、查询路径限流机制

#### 2.1 查询流程概览

```
HTTP 请求 → 并发控制 → 参数解析 → PromQL 解析 → 存储搜索 → 数据解压 → 函数执行 → 结果返回
```

#### 2.2 并发查询控制

**代码位置**: `app/vmselect/main.go`

```go
// Line 34-37: 配置定义
var (
    maxConcurrentRequests = flag.Int("search.maxConcurrentRequests", getDefaultMaxConcurrentRequests(),
        "The maximum number of concurrent search requests")
    maxQueueDuration = flag.Duration("search.maxQueueDuration", 10*time.Second,
        "The maximum time to wait in the queue when all concurrent requests are executed")
)

// Line 68: 并发控制 channel
var concurrencyLimitCh = make(chan struct{}, *maxConcurrentRequests)

// Line 121-158: 并发检查流程
func RequestHandler(w http.ResponseWriter, r *http.Request) bool {
    select {
    case concurrencyLimitCh <- struct{}{}:
        // 立即获得槽位
    default:
        // 槽位已满，等待
        t := time.NewTimer(*maxQueueDuration)
        select {
        case concurrencyLimitCh <- struct{}{}:
            t.Stop()
        case <-t.C:
            // 超时，返回 HTTP 429
            concurrencyLimitTimeout.Inc()
            http.Error(w, "too many concurrent requests", http.StatusTooManyRequests)
            return true
        }
    }
    defer func() { <-concurrencyLimitCh }()
    // 处理请求...
}
```

**相关指标**:
- `vm_concurrent_select_limit_reached_total`: 并发限制触发次数
- `vm_concurrent_select_limit_timeout_total`: 排队超时次数
- `vm_concurrent_select_current`: 当前并发查询数
- `vm_concurrent_select_capacity`: 最大并发容量

#### 2.3 查询超时控制

**代码位置**: `app/vmselect/searchutils/searchutils.go`

```go
// Line 18-20: 配置
var (
    maxQueryDuration = flag.Duration("search.maxQueryDuration", 30*time.Second,
        "The maximum duration for a query")
)

// Line 37-71: Deadline 计算
func GetDeadlineForQuery(r *http.Request, startTime time.Time) Deadline {
    dMax := maxQueryDuration.Milliseconds()
    // 支持 per-request timeout 参数覆盖
    timeout := httputils.GetDuration(r, "timeout", 0)
    if timeout <= 0 || timeout > dMax {
        timeout = dMax
    }
    return NewDeadline(startTime, timeout, "-search.maxQueryDuration")
}
```

**Deadline 检查点** (贯穿整个查询过程):
1. 存储搜索初始化: `netstorage.go:1126`
2. 遍历数据块: `netstorage.go:1188`
3. 数据解包: `netstorage.go:100-102`

```go
// netstorage.go:100-102
if rss.deadline.Exceeded() {
    tsw.mustStop.Store(true)
    return fmt.Errorf("timeout exceeded during query execution: %s", rss.deadline.String())
}
```

#### 2.4 查询长度限制

**代码位置**: `app/vmselect/prometheus/prometheus.go`

```go
// Line 37
var maxQueryLen = flagutil.NewBytes("search.maxQueryLen", 16*1024,
    "The maximum length for the query")

// Line 714-716: 检查
if len(query) > maxQueryLen.IntN() {
    return fmt.Errorf("too long query; got %d bytes; mustn't exceed `-search.maxQueryLen=%d` bytes",
        len(query), maxQueryLen.N)
}
```

#### 2.5 Series 数量限制

**代码位置**: `app/vmselect/prometheus/prometheus.go`

```go
// Line 50
var maxUniqueTimeseries = flag.Int("search.maxUniqueTimeseries", 300000,
    "The maximum number of unique time series the query can return")
```

在查询执行过程中会检查返回的 series 数量是否超限。

#### 2.6 采样点数量限制

**代码位置**: `app/vmselect/netstorage/netstorage.go`

```go
// Line 30-31
var (
    maxSamplesPerSeries = flag.Int("search.maxSamplesPerSeries", 30e6,
        "The maximum number of samples per series")
    maxSamplesPerQuery = flag.Int("search.maxSamplesPerQuery", 1e9,
        "The maximum number of samples per query")
)

// Line 1194-1200: 每查询采样点检查
samples += br.RowsCount()
if *maxSamplesPerQuery > 0 && samples > *maxSamplesPerQuery {
    return nil, fmt.Errorf("cannot select more than -search.maxSamplesPerQuery=%d samples; "+
        "either increase the flag value or reduce the query time range", *maxSamplesPerQuery)
}

// Line 472-476: 每 series 采样点检查
samples += len(upw.sb.Timestamps)
if *maxSamplesPerSeries > 0 && samples > *maxSamplesPerSeries {
    return fmt.Errorf("cannot process more than %d samples per series; "+
        "either increase -search.maxSamplesPerSeries or reduce time range", *maxSamplesPerSeries)
}
```

#### 2.7 并行 Worker 控制

**代码位置**: `app/vmselect/netstorage/netstorage.go`

```go
// Line 33
var maxWorkersPerQuery = flag.Int("search.maxWorkersPerQuery", defaultMaxWorkersPerQuery,
    "The maximum number of workers for parallel query processing")

// Line 186-191: Worker 数量计算
func MaxWorkers() int {
    n := *maxWorkersPerQuery
    if n <= 0 || n > gomaxprocs {
        n = gomaxprocs
    }
    return n
}
```

#### 2.8 返回结果点数限制

**代码位置**: `app/vmselect/prometheus/prometheus.go`

```go
// Line 58
var maxPointsPerTimeseries = flag.Int("search.maxPointsPerTimeseries", 30000,
    "The maximum points per timeseries to return")
```

超过此限制的数据点会被截断。

#### 2.9 查询内存限制

**代码位置**: `app/vmselect/promql/eval.go`

```go
// Line 36-43
var (
    maxMemoryPerQuery = flagutil.NewBytes("search.maxMemoryPerQuery", 0,
        "The maximum memory per query; if exceeded, the query is rejected")
    logQueryMemoryUsage = flagutil.NewBytes("search.logQueryMemoryUsage", 0,
        "Log queries that use more memory than this")
)
```

**内存限制器实现** (`promql/memory_limiter.go`):
```go
type memoryLimiter struct {
    MaxSize uint64
    mu      sync.Mutex
    usage   uint64
}

func (ml *memoryLimiter) Get(n uint64) bool {
    ml.mu.Lock()
    defer ml.mu.Unlock()
    if n > ml.MaxSize || ml.MaxSize-n < ml.usage {
        return false  // 超限，拒绝
    }
    ml.usage += n
    return true
}
```

#### 2.10 慢查询日志

**代码位置**: `app/vmselect/main.go`

```go
// Line 40
var logSlowQueryDuration = flag.Duration("search.logSlowQueryDuration", 5*time.Second,
    "Log queries with execution time exceeding this duration")
```

超过此时间的查询会被记录到日志，便于排查性能问题。

---

### 三、限流机制总览图

#### 写入路径限流层次

```
                     数据写入请求
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 1: 网络层保护                       │
    │  • HTTP ReadHeaderTimeout: 5s             │
    │  • HTTP IdleConnTimeout: 1min             │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 2: 并发控制                         │
    │  • maxConcurrentInserts: 2×CPU            │
    │  • insert.maxQueueDuration: 1min          │
    │  触发: HTTP 503                           │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 3: 请求大小限制                     │
    │  • maxInsertRequestSize: 32MB             │
    │  触发: 拒绝请求                           │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 4: 数据验证                         │
    │  • maxLabelsPerTimeseries: 30             │
    │  • maxLabelValueLen: 16KB                 │
    │  触发: 丢弃/截断标签                      │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 5: 基数限制 (关键!)                 │
    │  • maxHourlySeries: 可配置                │
    │  • maxDailySeries: 可配置                 │
    │  触发: 丢弃超限 series                    │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 6: 存储保护                         │
    │  • minFreeDiskSpaceBytes: 10MB            │
    │  • maxInmemoryParts: 60                   │
    │  触发: 只读模式 / 写入暂停                │
    └─────────────────────┬─────────────────────┘
                          │
                          ▼
                      数据落盘
```

#### 查询路径限流层次

```
                     查询请求
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 1: 并发控制                         │
    │  • maxConcurrentRequests: 基于CPU         │
    │  • maxQueueDuration: 10s                  │
    │  触发: HTTP 429                           │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 2: 请求验证                         │
    │  • maxQueryLen: 16KB                      │
    │  • maxQueryDuration: 30s                  │
    │  触发: 拒绝请求 / 超时中断                │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 3: 搜索范围控制                     │
    │  • maxUniqueTimeseries: 300000            │
    │  • maxTagKeys/Values: 100000              │
    │  触发: 返回错误                           │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 4: 数据量控制                       │
    │  • maxSamplesPerQuery: 1B                 │
    │  • maxSamplesPerSeries: 30M               │
    │  触发: 返回错误                           │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 5: 执行资源控制                     │
    │  • maxWorkersPerQuery: CPU核数            │
    │  • maxMemoryPerQuery: 可配置              │
    │  • Deadline 持续检查                      │
    │  触发: 限制并行度 / 拒绝请求              │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────▼─────────────────────┐
    │  Layer 6: 结果控制                         │
    │  • maxPointsPerTimeseries: 30000          │
    │  触发: 数据截断                           │
    └─────────────────────┬─────────────────────┘
                          │
                          ▼
                      返回结果
```

---

### 四、OOM 问题排查建议

当遇到 VictoriaMetrics OOM 时，可以按以下步骤排查:

#### 4.1 检查基数

访问 VMUI 的 Cardinality 页面: `/vmui/#/cardinality`

关注指标:
- 总 series 数量
- 高基数指标 (如 histogram bucket 类型)
- Label-value pairs 数量

#### 4.2 查看关键指标

```promql
# 写入限流触发
vm_concurrent_insert_limit_timeout_total
vm_hourly_series_limit_rows_dropped_total
vm_daily_series_limit_rows_dropped_total

# 查询限流触发
vm_concurrent_select_limit_timeout_total
vm_slow_queries_total

# 资源使用
process_resident_memory_bytes
vm_rows_read_per_query
vm_series_read_per_query
```

#### 4.3 推荐配置

对于 64GB 内存的机器，推荐配置:

```bash
# vmstorage
-memory.allowedPercent=60
-storage.maxHourlySeries=500000
-storage.maxDailySeries=2000000
-storage.minFreeDiskSpaceBytes=10737418240  # 10GB

# vmselect
-search.maxConcurrentRequests=8
-search.maxQueryDuration=60s
-search.maxUniqueTimeseries=500000
-search.maxMemoryPerQuery=4294967296  # 4GB
```

---

### 五、总结

VictoriaMetrics 实现了完善的多层限流保护机制:

1. **写入端**: 从并发控制、请求大小、标签验证、基数限制到磁盘空间保护，形成完整的写入保护链
2. **查询端**: 从并发控制、超时控制、数据量限制到内存限制，确保查询不会耗尽资源
3. **渐进式降级**: 限流策略采用排队→丢弃→只读的渐进式策略，尽可能保证服务可用性

合理配置这些限流参数，可以有效避免 OOM 问题，保障 VictoriaMetrics 的稳定运行。

---

## 参考资料

- [VictoriaMetrics 官方文档](https://docs.victoriametrics.com/)
- [VictoriaMetrics GitHub](https://github.com/VictoriaMetrics/VictoriaMetrics)
- 代码版本: v1.87.5+
