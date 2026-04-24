# Batch Processing

---

## Why Batch Size Matters

Without batching:
```sql
INSERT INTO post (title) VALUES ('A');  -- 1 roundtrip
INSERT INTO post (title) VALUES ('B');  -- 1 roundtrip
INSERT INTO post (title) VALUES ('C');  -- 1 roundtrip
-- 1000 inserts = 1000 roundtrips
```

With batching (batch_size=25):
```sql
INSERT INTO post (title) VALUES ('A'), ('B'), ..., ('Y');  -- 1 roundtrip for 25
-- 1000 inserts = 40 roundtrips
```

---

## Configuration

```properties
# Required for JDBC batching
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true   # batch same entity type together
spring.jpa.properties.hibernate.order_updates=true   # same for updates

# Verify batching is actually happening (check logs for "executing batch")
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.engine.jdbc.batch.internal.BatchingBatch=DEBUG
```

**Important:** `IDENTITY` generation strategy **completely disables** batching — Hibernate must get the generated ID back before it can move on. This is the primary reason to use `SEQUENCE`.

---

## Flush/Clear Pattern for Large Batches

Without flush/clear, the persistence context accumulates all entities in memory → OutOfMemoryError.

```java
@Transactional
public void importPosts(List<PostImportDTO> data) {
    final int BATCH_SIZE = 25;

    for (int i = 0; i < data.size(); i++) {
        PostImportDTO dto = data.get(i);
        Post post = new Post(dto.getTitle(), dto.getBody());
        entityManager.persist(post);

        if ((i + 1) % BATCH_SIZE == 0) {
            entityManager.flush();   // send batch to DB
            entityManager.clear();   // release all entities from first-level cache
        }
    }
    // final partial batch
    entityManager.flush();
    entityManager.clear();
}
```

**With @Transactional on outer method:** flush writes but does NOT commit — transaction commits at method end.

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

Don't load entities just to update a field:

```java
// ❌ WRONG — loads 10,000 entities, dirty-checks each, generates 10,000 UPDATEs
List<Post> posts = postRepository.findByStatus("DRAFT");
posts.forEach(p -> p.setStatus("ARCHIVED"));

// ✅ CORRECT — single SQL UPDATE
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Post p SET p.status = 'ARCHIVED' WHERE p.status = 'DRAFT' AND p.createdAt < :cutoff")
int archiveOldDrafts(@Param("cutoff") Instant cutoff);
```

```java
// ❌ WRONG — loads then deletes
List<Post> spam = postRepository.findByType("SPAM");
postRepository.deleteAll(spam);

// ✅ CORRECT — single DELETE
@Modifying(clearAutomatically = true)
@Query("DELETE FROM Post p WHERE p.type = 'SPAM'")
int deleteSpam();
```

**clearAutomatically = true** is critical: after a bulk DML, the first-level cache holds stale entities. Clearing it ensures subsequent reads go to DB.

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
