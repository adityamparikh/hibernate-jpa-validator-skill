# SQL Logging and Monitoring

Observability is the first step toward performance. You cannot optimize what you cannot see.

---

## Logging Layers (choose by environment)

| Tool | Environment | What It Shows | Bind Params | Timing |
|---|---|---|---|---|
| `hibernate.show_sql` | Dev only | Formatted SQL | ❌ | ❌ |
| SLF4J log levels | Dev/Test | SQL + params (separate lines) | ✅ | ❌ |
| **datasource-proxy** | Test | SQL + params + count assertions | ✅ | ✅ |
| **p6spy** | Dev/Staging | SQL + params + timing | ✅ | ✅ |
| **log4jdbc** | Dev/Staging | SQL + params | ✅ | ✅ |
| Hibernate Statistics | All envs | Aggregate counts, no SQL text | ❌ | ✅ avg |

---

## Option 1: Hibernate Built-In (Dev Only)

```properties
# application-dev.properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true  # adds /* load Post */ before each SQL

logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE  # bind parameters (Hibernate 6+)
```

**Hibernate 5 equivalent for bind params:**
```properties
logging.level.org.hibernate.type.descriptor.sql=TRACE
```

Sample output with `format_sql=true` and `use_sql_comments=true`:
```sql
/* load com.example.Post */
select
    p1_0.id,
    p1_0.title,
    p1_0.status
from
    post p1_0
where
    p1_0.id=?

binding parameter [1] as [BIGINT] - [42]
```

**Limitation:** Bind parameters log on a separate line — hard to correlate at scale. Use datasource-proxy or p6spy for complete statements.

---

## Option 2: datasource-proxy (Recommended for Tests)

datasource-proxy wraps your DataSource and intercepts every JDBC call. Ideal for asserting exact query counts in tests.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
    <version>1.10</version>
    <scope>test</scope>
</dependency>
```

### Test DataSource Config

```java
@Configuration
@Profile("test")
public class TestDataSourceConfig {

    @Bean
    DataSource dataSource(DataSourceProperties props) {
        HikariDataSource real = props.initializeDataSourceBuilder()
            .type(HikariDataSource.class).build();

        return ProxyDataSourceBuilder.create(real)
            .name("TestDS")
            .logQueryBySlf4j(SLF4JLogLevel.DEBUG)
            .multiline()
            .countQuery()                          // enables QueryCountHolder
            .build();
    }
}
```

### Asserting Query Count in Tests

```java
@SpringBootTest
class PostServiceTest {

    @Test
    void loadPostWithAuthor_shouldIssueOneQuery() {
        QueryCountHolder.clear();

        postService.getPostWithAuthor(1L);

        QueryCount count = QueryCountHolder.getGrandTotal();
        assertThat(count.getSelect()).isEqualTo(1);  // catch N+1 in CI
        assertThat(count.getTotal()).isEqualTo(1);
    }
}
```

This is the most powerful N+1 regression detection pattern — a CI test will fail the moment someone introduces a lazy-loaded access in a loop.

### Slow Query Threshold

```java
ProxyDataSourceBuilder.create(real)
    .logSlowQueryBySlf4j(200, TimeUnit.MILLISECONDS)  // warn on queries > 200ms
    .build();
```

---

## Option 3: p6spy (Dev/Staging, Zero Code Changes)

p6spy intercepts JDBC calls at the driver level. Shows complete SQL with bound parameters in a single log line — no correlation needed.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```properties
# application-dev.properties
spring.datasource.url=jdbc:p6spy:postgresql://localhost:5432/mydb
spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver
```

```properties
# spy.properties (in src/main/resources)
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
# Or for single-line with timing:
# logMessageFormat=%(executionTime)ms | %(sql)

# Exclude no-ops from logs
filter=true
exclude=SET,SHOW

# Slow query threshold (ms)
executionThreshold=200
```

Sample output:
```
1|7|2024-01-15 10:23:45|statement|SELECT id, title FROM post WHERE author_id = 42 AND status = 'ACTIVE'
```

The `1|7` prefix: connection id | execution time in ms.

---

## Option 4: Hibernate Statistics + Micrometer

For production observability. Statistics expose aggregate counts as metrics — no SQL text logged.

### Enable Statistics

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

### Expose via Micrometer (Spring Boot Actuator)

Spring Boot 2.3+ auto-registers Hibernate statistics as Micrometer metrics when `hibernate.generate_statistics=true` and `spring-boot-actuator` is on the classpath.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Available metrics (Prometheus format):
```
hibernate_sessions_open_total
hibernate_sessions_closed_total
hibernate_connections_obtained_total
hibernate_statements_prepared_total
hibernate_statements_executed_total
hibernate_entities_loads_total
hibernate_entities_inserts_total
hibernate_entities_updates_total
hibernate_entities_deletes_total
hibernate_collections_loads_total
hibernate_second_level_cache_hits_total
hibernate_second_level_cache_misses_total
hibernate_second_level_cache_puts_total
hibernate_query_executions_total
hibernate_query_execution_seconds{quantile="0.5"}   # slow query histogram
hibernate_query_execution_seconds{quantile="0.99"}
```

### Manual Statistics Access

```java
@Autowired EntityManagerFactory emf;

public HibernateHealthReport report() {
    Statistics s = emf.unwrap(SessionFactory.class).getStatistics();

    // Queries per session (N+1 smell if > 5-10)
    double avgQueriesPerSession = (double) s.getQueryExecutionCount() / s.getSessionOpenCount();

    // L2 cache effectiveness
    long hits = s.getSecondLevelCacheHitCount();
    long misses = s.getSecondLevelCacheMissCount();
    double hitRatio = hits + misses > 0 ? (double) hits / (hits + misses) : 0;

    return new HibernateHealthReport(avgQueriesPerSession, hitRatio,
        s.getSlowQueries());  // queries exceeding the slow query threshold
}
```

### Slow Query Logging via Statistics

```properties
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=200
```

Logs any HQL/SQL query taking > 200ms — no code changes, works in production.

---

## Recommended Setup by Environment

| Environment | Primary Tool | Secondary |
|---|---|---|
| **Unit/Integration tests** | datasource-proxy (query count assertions) | Hibernate statistics |
| **Local development** | p6spy (full SQL + timing) | `format_sql=true` for quick checks |
| **Staging** | p6spy or log4jdbc | Slow query threshold alerting |
| **Production** | Hibernate Statistics → Micrometer → Prometheus | Slow query log threshold |

**Never** use `show_sql=true` or `logging.level.org.hibernate.SQL=DEBUG` in production — too verbose and impacts performance.

---

## Slow Query Detection Summary

Three independent layers — use all three:

1. **`hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=200`** — logs slow JPQL/HQL queries
2. **p6spy `executionThreshold=200`** — logs slow raw JDBC calls (including native SQL)
3. **Database-side `pg_stat_statements`** / MySQL `slow_query_log` — catches everything, even non-Hibernate queries
