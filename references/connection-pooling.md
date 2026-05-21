# Connection Pooling

HikariCP is the default and best pool for Spring Boot. These settings apply to Spring Boot 3.x + HikariCP 5.x.

---

## Complete Configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      # Pool sizing
      maximum-pool-size: 10          # see sizing formula below
      minimum-idle: 10               # = maximum (fixed pool, HikariCP recommendation)

      # Timeouts
      connection-timeout: 30000      # 30s: max wait for connection from pool
      idle-timeout: 600000           # 10m: remove idle connections (only when min < max)
      max-lifetime: 1800000          # 30m: rotate connections (must be < DB wait_timeout)

      # Leak detection
      leak-detection-threshold: 2000 # warn if connection held > 2s

      # Validation
      connection-test-query: SELECT 1  # only if JDBC4 driver doesn't support isValid()
      keepalive-time: 30000           # send keepalive to prevent firewall timeout

      # Performance
      auto-commit: false              # let Spring @Transactional manage commits
      pool-name: HikariPool-MyApp    # named pool for metrics
```

---

## Pool Sizing Formula

**Formula (Hyun-Joo's rule of thumb):**
```
pool_size = (core_count * 2) + effective_spindle_count
```

For a 4-core machine with SSD (effective_spindle_count ≈ 1):
```
pool_size = (4 * 2) + 1 = 9 → round to 10
```

**Counterintuitive truth:** More connections ≠ more throughput.
- Each connection is a thread on the DB server
- Excessive connections cause context switching overhead
- For PostgreSQL on a 16-core machine: 30–50 connections maximum
- For connection-intensive apps behind a proxy: use PgBouncer between app and DB

**Setting `minimum-idle = maximum-pool-size`** (fixed pool):
- Avoids connection acquisition latency at burst time
- Avoids noisy tear-down/re-creation cycles
- HikariCP recommends this explicitly

---

## Connection Validation

HikariCP auto-validates via JDBC4 `isValid()` — only set `connection-test-query: SELECT 1` for pre-JDBC4 drivers. On MySQL, keep `max-lifetime` below `wait_timeout` (default 8h, usually fine). On PostgreSQL behind load balancers, set `keepalive-time` to prevent idle-timeout drops.

---

## Statement Caching

HikariCP delegates statement caching to the JDBC driver. The driver maintains a per-connection LRU cache of `PreparedStatement` objects — parsing SQL is skipped on cache hits.

### PostgreSQL (server-side prepared statements)

```properties
spring.datasource.hikari.data-source-properties.prepStmtCacheSize=250       # how many statements to cache per connection
spring.datasource.hikari.data-source-properties.prepStmtCacheSqlLimit=2048  # max SQL length eligible for caching
```

PostgreSQL auto-prepares statements on the server after 5 executions. The driver caches the prepared statement handles client-side.

### MySQL / MariaDB (critical — disabled by default)

```properties
spring.datasource.hikari.data-source-properties.cachePrepStmts=true         # MUST enable — off by default!
spring.datasource.hikari.data-source-properties.prepStmtCacheSize=250
spring.datasource.hikari.data-source-properties.prepStmtCacheSqlLimit=2048
spring.datasource.hikari.data-source-properties.useServerPrepStmts=true     # send to MySQL server for parsing
spring.datasource.hikari.data-source-properties.rewriteBatchedStatements=true  # enables true JDBC batching on MySQL
```

`rewriteBatchedStatements=true` is **required** for JDBC batching to work on MySQL — without it, MySQL executes each statement individually even with `executeBatch()` called.

### Impact

A well-tuned statement cache reduces CPU overhead on both client and server, especially for applications issuing thousands of similar parameterized queries per second (e.g., `SELECT * FROM post WHERE id = ?`).

---

## Leak Detection

```yaml
hikari:
  leak-detection-threshold: 2000  # ms — log stack trace if connection held > 2s
```

A leak is logged at WARN level with the stack trace showing where the connection was acquired. Common causes:
- Missing `@Transactional` on a method that calls JPA
- Transaction not committed/rolled back in error path
- Long-running operations holding a connection (HTTP call inside transaction)

**Best practice:** Never hold a DB connection while doing network I/O. Fetch data, close transaction, then call external services.

---

## Metrics and Monitoring

HikariCP exposes metrics via Micrometer (auto-configured in Spring Boot):

```java
// Metric names (in Prometheus format):
// hikaricp.connections.active       - currently in use
// hikaricp.connections.idle         - waiting in pool
// hikaricp.connections.pending      - threads waiting for a connection
// hikaricp.connections.timeout.total - how many timed out
// hikaricp.connections.acquire      - histogram of wait time
// hikaricp.connections.usage        - histogram of hold time
```

**Alert thresholds:**
- `connections.active / maximum-pool-size > 0.9` → near-saturation, increase pool
- `connections.pending > 0` persistent → pool too small
- `connections.timeout.total` increasing → investigate long-held connections
- `connections.acquire` p99 > 100ms → pool saturation or slow connection init

---

## Common Pool Exhaustion Causes

| Cause | Symptom | Fix |
|---|---|---|
| Long transactions | `connections.usage` p99 high | Break up long TX, avoid network I/O inside TX |
| Missing `@Transactional` rollback | Connection not returned on exception | Use `@Transactional` or explicit try-finally |
| Nested transactions without propagation | Each nested call opens new connection | Check PROPAGATION_REQUIRED vs REQUIRES_NEW |
| Thread pool mismatch | Web thread pool >> HikariCP pool | Align sizes or use REQUIRES_NEW carefully |
| Slow queries | Connections held waiting for DB | Add indexes, optimize queries |

---

## FlexyPool — Adaptive Pool Sizing and Monitoring

HikariCP is excellent but has a fixed maximum pool size. FlexyPool wraps any connection pool and provides:
- **Adaptive pool sizing** — automatically grows the pool when connections are in high demand, shrinks it when idle
- **Rich metrics** — acquisition time histograms, overflow counts, timeout rates
- **Pool exhaustion early warning** — alerts before `connectionTimeout` fires

```xml
<dependency>
    <groupId>com.vladmihalcea.flexy-pool</groupId>
    <artifactId>flexy-hikaricp</artifactId>
    <version>2.2.3</version>
</dependency>
```

```java
@Bean
public FlexyPoolDataSource<HikariDataSource> flexyPoolDataSource(
        HikariDataSource hikariDataSource) {

    FlexyPoolConfiguration<HikariDataSource> config =
        new FlexyPoolConfiguration.Builder<>(
            "myApp-pool",
            hikariDataSource,
            HikariCPPoolAdapter.FACTORY
        )
        .setMetricsFactory(new MicrometerMetricsFactory())  // → Prometheus
        .setConnectionAcquisitionTimeThreshold(50L)         // warn if > 50ms to get connection
        .build();

    return new FlexyPoolDataSource<>(config,
        new IncrementPoolOnTimeoutConnectionAcquisitionStrategy.Factory<>(5),  // grow by 5 on timeout
        new RetryConnectionAcquisitionStrategy.Factory<>(2)                    // retry 2x before timeout
    );
}
```

**FlexyPool metrics (Micrometer):**
- `flexypool.connection.acquire` — histogram of time to get a connection
- `flexypool.connection.lease` — histogram of time connection was held
- `flexypool.connection.overflow` — connections acquired beyond pool max

**When to add FlexyPool:**
- Production systems where pool size is hard to right-size in advance
- High-traffic apps with bursty connection demand
- When you want connection acquisition time SLAs in Grafana

---

## Multiple Data Sources

Standard pattern for read/write split: define `writeDataSource` (`@Primary`) and `readDataSource` `@Bean`s with `@ConfigurationProperties("spring.datasource.write.hikari")` / `...read.hikari`. The read pool can be smaller — read queries are typically faster and less contended. For routing logic, see `references/spring-transactions.md` (`AbstractRoutingDataSource` + `LazyConnectionDataSourceProxy`).
