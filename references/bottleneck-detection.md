# Bottleneck Detection Pattern Library

## Pattern Recognition Framework

For each pattern: detection signal, mechanism, severity assessment, and concrete fix.

---

## 1. N+1 Database Queries

**Detection in Tideways:**
- Slow SQL page shows same query pattern repeated many times (10x-1000x)
- Transaction with SQL prefix where query count >> result rows
- Stacktrace shows individual record loads inside iteration

**Mechanism:**
```php
foreach ($orders as $order) {
    $user = $userRepository->findById($order->getUserId());
    // Each iteration fires: SELECT * FROM users WHERE id = ?
}
// 100 orders = 100 individual queries instead of 1 bulk query
```

**Severity Assessment:**
- Query count x avg query time = total SQL overhead
- >50 queries of same pattern = definite N+1
- Growing with data volume = will get worse

**Fix:**
```php
// Collect IDs first, then bulk load
$userIds = array_map(fn($order) => $order->getUserId(), $orders);
$users = $userRepository->findByIds($userIds);

foreach ($orders as $order) {
    $user = $users[$order->getUserId()] ?? null;
}
```

**Confidence:** HIGH when query count correlates with collection size.

---

## 2. Lock Contention

**Detection in Tideways:**
- SESSION prefix on transactions
- Multiple concurrent requests showing high wall time but low CPU time
- Intermittent extreme slowness (>10x normal) on otherwise fast endpoints
- AJAX endpoints blocking each other

**Mechanism:**
PHP file sessions lock the session file. Concurrent requests from same user serialize:
```
Request A: session_start() → lock acquired → processing (2s) → session_write_close()
Request B: session_start() → WAITING FOR LOCK → processing (0.5s) → done
Request B total time: 2.5s (2s waiting + 0.5s actual work)
```

**Severity Assessment:**
- Affects user-perceived performance directly
- Worse with more AJAX calls on page
- Multiplies with page complexity

**Fix:**
1. Use Redis sessions with locking disabled for read-only endpoints
2. Call `session_write_close()` early in controllers that don't modify session
3. Reduce concurrent AJAX requests from frontend
4. Batch multiple API calls into single request

**Confidence:** HIGH when wall time >> CPU time and SESSION prefix present.

---

## 3. Repetitive Object Creation

**Detection in Tideways:**
- COMPILE prefix despite OPcache being enabled
- Trace shows factory or constructor calls deep in iteration loops
- Memory growing linearly with iteration count
- `__construct()` appearing many times in trace for same class

**Mechanism:**
```php
foreach ($items as $item) {
    // Creates new heavy object every iteration
    $processor = new HeavyProcessor($config, $logger, $validator);
    $processor->process($item);
}
```

**Severity Assessment:**
- Object creation cost: ~0.1-1ms each
- x1000 iterations = 100ms-1s overhead
- Also increases memory usage and GC pressure

**Fix:**
- Reuse objects when stateless: instantiate once before the loop
- Use dependency injection with shared instances instead of `new` in loops
- For stateful objects: reset state instead of recreating

**Confidence:** MEDIUM - need trace detail to confirm creation count.

---

## 4. Middleware/Hook Chain Overhead

**Detection in Tideways:**
- Deep call stacks for simple operations
- Multiple wrapper/decorator layers visible in trace
- Framework hook system executing many listeners per event
- Call depth >20 for straightforward operations

**Mechanism:**
```
handleRequest()                          # 1ms actual work
  → Middleware1::process()               # wraps everything, adds 5ms
    → Middleware2::process()             # wraps everything, adds 3ms
      → Middleware3::process()           # wraps everything, adds 2ms
        → Controller::execute()          # actual execution
      → Middleware3 post-logic           # 1ms
    → Middleware2 post-logic             # 2ms
  → Middleware1 post-logic               # 3ms
# Total: 17ms for 1ms of actual work
```

**Severity Assessment:**
- Each middleware/hook layer adds minimum ~0.5ms overhead
- On hot paths (called 100x per request), this multiplies
- Watch for hooks on frequently-called methods (ORM load, template render, routing)

**Fix:**
1. Reduce middleware stack: remove unnecessary layers
2. Add early-return conditions: `if (!$this->isApplicable()) return $next($request);`
3. Merge multiple related middleware into single handler
4. Review if each hook/listener is still needed

**Confidence:** HIGH when trace shows deep nesting on hot path.

---

## 5. Too Many Database Roundtrips

**Detection in Tideways:**
- SQL prefix with many small queries instead of few large ones
- Total query count per request >100
- Many similar queries with different parameters

**Mechanism:**
Different from N+1: multiple distinct queries that could be combined or eliminated.
```sql
-- 3 separate queries that could be 1
SELECT * FROM settings WHERE key = 'site.name';
SELECT * FROM settings WHERE key = 'site.url';
SELECT * FROM settings WHERE key = 'site.theme';
```

**Fix:**
1. Check if configuration/settings are being cached properly
2. Batch related queries into single query with `IN ()` clause
3. Preload related data with JOINs or eager loading
4. Verify application-level cache is enabled and working

**Confidence:** MEDIUM - need to verify queries are actually redundant.

---

## 6. IO Wait Isolation

**Detection in Tideways:**
- High wall time, low CPU time in trace
- External HTTP calls showing >500ms
- File operations on NFS/remote storage
- Redis/cache operations with high latency

**Mechanism:**
PHP blocks on IO operations. The thread is idle but the request is slow.

**Classification:**

| IO Type | Signal | Typical Cause |
|---------|--------|--------------|
| Database IO | SQL prefix, query time in Slow SQLs | Missing index, table lock, slow disk |
| External HTTP | HTTP prefix, external call in trace | Slow API, DNS resolution, timeout config |
| File IO | High wall time on `include()` or `file_get_contents()` | NFS latency, missing OPcache |
| Session IO | SESSION prefix | File sessions, Redis latency |
| Cache IO | High cache operation time | Redis overloaded, network hop |

**Fix by type:**
- **Database:** Index optimization, query optimization, read replicas
- **External HTTP:** Async calls, queue processing, caching responses, timeouts
- **File IO:** OPcache, local disk, reduce file operations
- **Session IO:** Redis sessions, reduce session data
- **Cache IO:** Local Redis, pipeline commands, reduce cache calls

**Confidence:** HIGH when wall time to CPU time ratio >5:1.

---

## 7. Cache Stampede

**Detection in Tideways:**
- Periodic spikes in response time
- Low Cache Hit Rate in Performance Overview
- Spikes coincide with cache TTL expiry
- Multiple concurrent requests for same expensive operation

**Mechanism:**
Cache expires → N requests simultaneously try to regenerate → all hit database → thundering herd.

**Severity Assessment:**
- Proportional to traffic volume
- Worse with short TTL and expensive regeneration
- Can cascade into database overload

**Fix:**
1. **Lock-based regeneration**: First request acquires lock, others wait or serve stale
2. **Probabilistic early expiry**: Regenerate before TTL expires
3. **Background regeneration**: Cron job pre-warms cache before expiry
4. **Stale-while-revalidate**: Serve stale cache while regenerating in background

**Confidence:** MEDIUM - need correlation between cache expiry and spikes.

---

## 8. Cold Application Bootstrap

**Detection in Tideways:**
- First request after deploy much slower (>3x)
- COMPILE prefix on first request only
- Subsequent requests normal speed
- Affects all transactions equally

**Mechanism:**
Application framework must compile class maps, load configuration, and build object graph on first request after cache clear.

**Severity Assessment:**
- Only affects first request(s) after deploy/cache clear
- Can be significant on large applications with many classes
- Affects all PHP-FPM workers independently

**Fix:**
1. Pre-compile/warm application cache during deployment (before switching traffic)
2. OPcache preloading (`opcache.preload`)
3. Warm-up script that hits key URLs after deploy
4. Zero-downtime deployment (new code warm, then switch)

**Confidence:** HIGH when first request is consistently >3x slower.

---

## 9. Memory Growth / Leak

**Detection in Tideways:**
- Memory column in transaction table growing over time
- Individual requests with unusually high memory
- Memory spike in specific transaction types
- OOM kills in error logs

**Mechanism:**
Objects not freed during request lifecycle. Common in:
- Large dataset loads without pagination
- Event listeners accumulating data in properties
- Circular references preventing garbage collection
- Static properties accumulating across requests (in long-running processes)

**Severity Assessment:**
- >256MB per request: investigate
- >400MB per request: critical, will cause OOM
- Growing per request in CLI: memory leak in long-running process

**Fix:**
1. **Pagination**: Load data in chunks instead of all at once
2. **Clear references**: Unset large arrays/objects after processing
3. **Check event listeners**: Listeners that store data in class properties accumulate
4. **For CLI/workers**: Set max iterations, restart processes periodically

**Confidence:** MEDIUM-HIGH based on memory trend data.

---

## 10. Serialization Overhead

**Detection in Tideways:**
- `json_encode()` / `json_decode()` / `serialize()` / `unserialize()` in trace
- High CPU time relative to actual work
- Large payloads being converted

**Mechanism:**
Serialization of large objects (e.g., full entity data, complex nested structures) is CPU-intensive.

**Severity Assessment:**
- >10ms per serialization call on large objects
- Worse with nested objects and circular references
- Multiplied by frequency per request

**Fix:**
1. Serialize only needed fields, not entire objects
2. Use lighter formats (JSON vs PHP serialize)
3. Cache serialized output instead of re-serializing
4. Reduce payload size at source

**Confidence:** LOW-MEDIUM - need trace detail to confirm.

---

## Anomaly Detection Heuristics

When analyzing Tideways data, look for these anomaly patterns:

### Response Time Anomalies
| Pattern | Likely Cause |
|---------|-------------|
| Sudden jump in p95 after deploy | New code regression |
| Gradual increase over days/weeks | Data growth, index degradation |
| Periodic spikes every N minutes | Cron job interference |
| Spikes at specific hours | Traffic pattern + undersized infrastructure |
| Random extreme outliers (>10x avg) | GC pause, external service timeout, lock wait |

### Memory Anomalies
| Pattern | Likely Cause |
|---------|-------------|
| Step increase after deploy | New feature loading more data |
| Gradual growth over days | Accumulating cached data, growing sessions |
| Spikes on specific transactions | Large dataset loads, PDF generation |
| Consistent high across all transactions | Base memory usage too high, too many modules |

### Error Rate Anomalies
| Pattern | Likely Cause |
|---------|-------------|
| Spike coinciding with deploy | Regression in new code |
| Gradual increase | External service degradation, data issues |
| Correlates with traffic spike | Resource exhaustion under load |
| Only specific transaction | Isolated bug in that code path |

### Traffic vs Latency Correlation
| Pattern | Meaning |
|---------|---------|
| Latency constant regardless of traffic | CPU-bound or code issue |
| Latency increases linearly with traffic | Resource contention (DB connections, locks) |
| Latency jumps at traffic threshold | Worker pool exhaustion, connection limits |
| Latency spikes during low traffic | Background jobs competing for resources |
