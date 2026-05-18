# Batch Processing

---

## Why Batch Size Matters

Without batching, 1000 inserts = 1000 roundtrips. With `batch_size=25`, the JDBC driver groups 25 inserts per network call → 40 roundtrips. Latency drops near-linearly with batch size up to the network/driver sweet spot (~25–50 for most databases).

---

## Configuration

```properties
# Required for JDBC batching
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true   # batch same entity type together
spring.jpa.properties.hibernate.order_updates=true   # same for updates

# Required when using @Version (optimistic locking) with batching:
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true

# Verify batching is actually happening (check logs for "executing batch")
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.engine.jdbc.batch.internal.BatchingBatch=DEBUG
```

**`hibernate.jdbc.batch_versioned_data=true`:** By default, Hibernate will not batch `UPDATE` statements for versioned entities (those with `@Version`) because it cannot know how many rows were affected per statement in a batch — and it needs that to detect optimistic locking violations. Setting this to `true` tells Hibernate to trust the JDBC driver to report per-statement row counts, enabling batching for versioned entities. Supported by all modern drivers (PostgreSQL, MySQL 5.1.18+, Oracle 12c+).

**Important:** `IDENTITY` generation strategy **completely disables** batching — Hibernate must get the generated ID back before it can move on. This is the primary reason to use `SEQUENCE`.

---

## Flush/Clear Pattern for Large Batches

```java
@Transactional
public void importPosts(List<PostImportDTO> data) {
    for (int i = 0; i < data.size(); i++) {
        entityManager.persist(new Post(data.get(i).getTitle(), data.get(i).getBody()));
        if ((i + 1) % 25 == 0) {
            entityManager.flush();   // ship the batch
            entityManager.clear();   // empty L1 cache so it doesn't grow unbounded
        }
    }
    entityManager.flush();           // final partial batch
    entityManager.clear();
}
```

`flush()` ships SQL to the DB but does NOT commit — the outer `@Transactional` commits at method exit. Without `clear()`, the L1 cache holds every persisted entity until commit → OOM at scale.

---

## StatelessSession — Bypass the First-Level Cache

For pure bulk import where you never need to navigate associations:

```java
@Autowired
private SessionFactory sessionFactory;

public void bulkImport(List<PostImportDTO> data) {
    try (StatelessSession session = sessionFactory.openStatelessSession()) {
        Transaction tx = session.beginTransaction();
        int count = 0;

        for (PostImportDTO dto : data) {
            Post post = new Post(dto.getTitle(), dto.getBody());
            session.insert(post);  // no first-level cache, no dirty checking

            if (++count % 25 == 0) {
                session.getTransaction().commit();
                tx = session.beginTransaction();
            }
        }
        tx.commit();
    }
}
```

**StatelessSession limitations:**
- No first-level cache
- No lazy loading
- No cascading
- No event listeners (no auditing, no @PrePersist)

Use only for raw data import pipelines where those features aren't needed.

---

## Bulk UPDATE/DELETE with JPQL

Don't load entities just to update or delete them. Use `@Modifying` + JPQL `UPDATE`/`DELETE`:

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Post p SET p.status = 'ARCHIVED' WHERE p.status = 'DRAFT' AND p.createdAt < :cutoff")
int archiveOldDrafts(@Param("cutoff") Instant cutoff);
```

`clearAutomatically = true` is critical — after bulk DML, the L1 cache holds pre-DML entities; clearing forces subsequent reads to hit DB. `flushAutomatically = true` flushes pending changes first so they aren't overwritten by the bulk op.

---

## Spring @Transactional Chunk Pattern

For batch jobs processing large datasets:

```java
@Service
public class PostArchiver {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private PostArchiver self;  // inject self for @Transactional proxy

    // Non-transactional outer method — iterates in chunks
    public void archiveAll() {
        int page = 0;
        int pageSize = 500;
        List<Long> ids;

        do {
            ids = postRepository.findDraftIdsBefore(cutoff, PageRequest.of(page++, pageSize));
            if (!ids.isEmpty()) {
                self.archiveChunk(ids);  // call via proxy for transaction boundary
            }
        } while (ids.size() == pageSize);
    }

    // Each chunk runs in its own transaction — one commit per 500 rows
    @Transactional
    public void archiveChunk(List<Long> ids) {
        postRepository.updateStatusByIds("ARCHIVED", ids);
    }
}
```

**Why separate transactions per chunk:**
- Single giant transaction = holds DB locks for entire duration
- Failed chunk can be retried without rolling back all previous work

---

## Verifying Batching with p6spy

Add p6spy to see exactly what JDBC sends:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```properties
# application.properties
spring.datasource.url=jdbc:p6spy:postgresql://localhost:5432/mydb
spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver
```

```properties
# spy.properties
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
```

Batch execution appears as:
```
batch: 25 statements executed
  1. INSERT INTO post (id, title, body) VALUES (50, ...)
  2. INSERT INTO post (id, title, body) VALUES (51, ...)
  ...
```

If you see individual INSERTs (not batch), check: IDENTITY generator, missing batch_size config, or entity hierarchy (TABLE_PER_CLASS disables batching).

---

## Cursor-Based Processing with ScrollableResults

For processing very large result sets without loading everything into memory:

```java
@Transactional(readOnly = true)
public void processAllPosts() {
    Session session = entityManager.unwrap(Session.class);

    // ScrollableResults fetches rows from DB cursor — constant memory regardless of table size
    try (ScrollableResults<Post> scroll = session.createQuery(
            "FROM Post p WHERE p.status = 'ACTIVE'", Post.class)
            .setFetchSize(50)          // fetch 50 rows at a time from DB cursor
            .scroll(ScrollMode.FORWARD_ONLY)) {

        int count = 0;
        while (scroll.next()) {
            Post post = scroll.get();
            processPost(post);

            if (++count % 50 == 0) {
                session.flush();
                session.clear();   // prevent L1 cache growing unbounded
            }
        }
    }
}
```

**When to use ScrollableResults vs batch paging:**
- `ScrollableResults` — continuous streaming with a live DB cursor. Best for ETL pipelines, exports, or when you need to process ALL rows in order.
- Chunk-by-page — closes and re-opens the query per chunk. Better when rows may change between chunks, or when you need restartable processing.

**`setFetchSize(50)`** is critical — without it, some JDBC drivers (PostgreSQL) load the entire result set into client memory before returning the first row.
