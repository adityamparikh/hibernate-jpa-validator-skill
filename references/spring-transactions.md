# Spring Transactions and Concurrency Control

Transactions are where most JPA correctness bugs hide. Get the boundary, propagation, and isolation right — then worry about performance.

---

## @Transactional Boundaries

### Rule: Service Layer Owns the Transaction

```java
// ❌ WRONG — @Transactional on the repository — too granular
public interface PostRepository extends JpaRepository<Post, Long> {
    @Transactional  // each call is its own tx — N+1 becomes N+1 transactions
    List<Post> findAll();
}

// ✅ CORRECT — service method opens the transaction; repository inherits it
@Service
public class PostService {

    @Transactional(readOnly = true)
    public List<PostDTO> getActivePosts() {
        return postRepository.findByStatus("ACTIVE").stream()
            .map(this::toDTO)
            .toList();
    }
}
```

`JpaRepository` methods are already `@Transactional` internally — explicit annotation only matters at the layer that defines the unit of work.

### Rule: Always Use `readOnly = true` for Reads

```java
@Transactional(readOnly = true)
public List<PostSummary> findSummaries() { ... }
```

What `readOnly = true` actually does:
1. Hibernate sets `FlushMode.MANUAL` — no auto-flush at query boundaries
2. Skips dirty-check snapshot creation — saves heap on large result sets
3. Spring sets the JDBC connection to read-only (driver-dependent — PostgreSQL honors this; MySQL ignores it)
4. **Does NOT prevent writes** — entities loaded are still mutable

### Anti-Pattern: @Transactional on Public Constructors / Self-Calls

```java
@Service
public class PostService {

    public void process(Long id) {
        doWork(id);  // ❌ Spring AOP bypassed — internal call goes straight to method, no transaction
    }

    @Transactional
    public void doWork(Long id) { ... }
}
```

Spring's transaction proxy only wraps **external** calls. Two fixes:
1. Inject `self` (`@Lazy PostService self`) and call `self.doWork(id)`.
2. Move `doWork` to a separate bean.

---

## Propagation

| Propagation | When to Use |
|---|---|
| `REQUIRED` (default) | Joins existing tx or starts new — almost always correct |
| `REQUIRES_NEW` | Audit logs, error logging — must commit independently of outer tx |
| `MANDATORY` | Internal helper that must NEVER run without an enclosing tx |
| `SUPPORTS` | Read methods that work both in and out of a tx |
| `NEVER` / `NOT_SUPPORTED` | Rare — bypass tx for performance-sensitive read |
| `NESTED` | Savepoints — only if your driver supports them; usually avoid |

### REQUIRES_NEW Gotcha

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    auditService.logOrder(order);  // calls @Transactional(propagation = REQUIRES_NEW)
}

// auditService method:
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logOrder(Order o) { ... }
```

**Two physical connections used simultaneously** during this call. If your pool size is 10 and 10 requests hit `placeOrder` at once, you need 20 connections — pool starvation deadlock.

→ See `references/connection-pooling.md` for sizing implications.

---

## Isolation Levels

| Level | Prevents | Default In |
|---|---|---|
| `READ_UNCOMMITTED` | Nothing useful | Never use |
| `READ_COMMITTED` | Dirty reads | PostgreSQL, Oracle, SQL Server |
| `REPEATABLE_READ` | Dirty + non-repeatable reads | MySQL InnoDB |
| `SERIALIZABLE` | All anomalies | Never default — heavy locking / SSI conflicts |

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public BigDecimal computeReportTotal(Long reportId) {
    // Need consistent reads across multiple queries within the tx
}
```

**Most application code should use the database default.** Only override when you have a specific anomaly to prevent.

---

## Locking

### Optimistic Locking (default — strongly recommended)

```java
@Entity
public class Account {
    @Id Long id;

    @Version
    private Long version;  // Hibernate adds WHERE version = ? to UPDATEs

    private BigDecimal balance;
}
```

On UPDATE, Hibernate generates:
```sql
UPDATE account SET balance = ?, version = version + 1
WHERE id = ? AND version = ?
```

If `rowsAffected == 0` → `OptimisticLockException`. Application retries.

### Pessimistic Locking (only when you must)

```java
// SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findForUpdate(@Param("id") Long id);

// SELECT ... FOR SHARE (read lock)
@Lock(LockModeType.PESSIMISTIC_READ)
Account findShared(Long id);
```

Use only when:
- You cannot retry on conflict (e.g., calling external system)
- You need to prevent concurrent reads from making decisions on stale data

**Always set a timeout** to prevent indefinite waits:
```java
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
```

### Lock Skipping (PostgreSQL)

```java
// Skip rows already locked — useful for queue-style workers
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))  // SKIP LOCKED
@Query("SELECT j FROM Job j WHERE j.status = 'PENDING' ORDER BY j.id")
List<Job> claimNext(Pageable pageable);
```

---

## LazyConnectionDataSourceProxy

By default, Spring acquires a JDBC connection at the **start** of a `@Transactional` method, even if no SQL runs. For read-heavy services with many no-op transactions, this wastes pool capacity.

```java
@Bean
@Primary
public DataSource dataSource(@Qualifier("realDataSource") DataSource real) {
    return new LazyConnectionDataSourceProxy(real);
}
```

Now connections are acquired only on first SQL call. Combine with the connection pool to dramatically increase effective capacity for services that often short-circuit (cache hits, validation rejection).

**Trade-off:** isolation level / read-only flag must be deferred until first statement — driver behavior varies. Test under load.

---

## Read-Write Routing

```java
@Configuration
public class RoutingDataSourceConfig {

    @Bean
    public DataSource routingDataSource(
            @Qualifier("primary") DataSource primary,
            @Qualifier("replica") DataSource replica) {

        AbstractRoutingDataSource router = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "REPLICA" : "PRIMARY";
            }
        };
        router.setTargetDataSources(Map.of(
            "PRIMARY", primary,
            "REPLICA", replica
        ));
        router.setDefaultTargetDataSource(primary);
        return new LazyConnectionDataSourceProxy(router);  // lazy is essential here
    }
}
```

`@Transactional(readOnly = true)` methods route to the replica; writes go to primary. The `LazyConnectionDataSourceProxy` wrapper is required because `determineCurrentLookupKey()` runs at first connection acquisition, after Spring has set the read-only flag.

**Replication lag caveat:** A user who just submitted a write may not see their own change on the replica. Either pin specific endpoints to primary, or use session-affinity hints.

---

## Transaction Synchronization

Run code at commit/rollback time:

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);

    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                eventPublisher.publishEvent(new OrderPlacedEvent(order.getId()));
            }
        }
    );
}
```

Cleaner with Spring's `@TransactionalEventListener`:

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order.getId()));
    }
}

@Component
public class OrderEmailListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent e) {
        emailService.send(e);  // only runs if tx committed
    }
}
```

Critical for: external API calls, message publishing, cache invalidation — anything you don't want to run if the DB tx rolls back.

---

## Common Anti-Patterns

| Symptom | Root Cause | Fix |
|---|---|---|
| `LazyInitializationException` outside service layer | Tx ended before view rendered | OSIV off + DTO projection in service |
| Tx works in tests, fails in prod | Self-invocation bypassing AOP | Inject `self` or split classes |
| `OptimisticLockException` floods | Hot row, no retry strategy | Use `@Retryable` with backoff |
| Connection pool exhausted under load | Long-running tx + external API call | Move external calls outside tx |
| Stale reads after `@Modifying` | First-level cache not cleared | `clearAutomatically = true` |
| Replica returns stale data | Read-after-write on routed DS | Pin endpoint to primary |

---

## Open Session In View (OSIV)

Spring Boot defaults `spring.jpa.open-in-view=true`. This keeps the Hibernate session open for the entire HTTP request — convenient, but:
- Hides lazy-loading in the view layer (silent N+1)
- Holds a database connection for the response duration
- Masks transaction boundary issues during testing

**Recommendation: turn it off.**

```properties
spring.jpa.open-in-view=false
```

Then service methods MUST return DTOs (not entities with lazy associations) — which is the correct architecture anyway.
