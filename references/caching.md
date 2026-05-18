# Second-Level Caching

---

## Cache Layers in Hibernate

| Layer | Scope | What's Cached |
|---|---|---|
| First-level (L1) | Single Session | Always on — entities loaded in current tx |
| Second-level (L2) | SessionFactory (app-wide) | Entity state, collections, query results |
| Query cache | SessionFactory | Query result IDs (references L2 for entities) |

---

## Concurrency Strategies

Choose based on your read/write profile:

| Strategy | When to Use | Behavior |
|---|---|---|
| `READ_ONLY` | Reference data, immutable entities | No lock, no copy-on-write. Fastest. Throws if update attempted. |
| `NONSTRICT_READ_WRITE` | Mostly read, rare updates, stale ok for brief window | Evicts on update, brief inconsistency window possible |
| `READ_WRITE` | Concurrent reads and writes, correctness required | Soft locks during update, consistent reads |
| `TRANSACTIONAL` | JTA transactions, strict isolation | XA-capable provider required (Infinispan) |

---

## Entity Cache Configuration

```java
// Hibernate 6+ annotation
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Post { ... }

// Reference data — always READ_ONLY + @Immutable
@Entity
@Immutable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country { ... }

// JPA 2.2+ alternative (works with any provider)
@Entity
@jakarta.persistence.Cacheable
public class Post { ... }
```

Enable L2 cache in Spring Boot:
```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.internal.JCacheRegionFactory
spring.cache.jcache.config=classpath:ehcache.xml
```

---

## Collection Cache

Collections need a separate cache entry:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Post {
    @OneToMany(mappedBy = "post")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // cache the collection too
    private List<Comment> comments;
}
```

Without `@Cache` on the collection, loading `post.getComments()` always hits the DB even if the Post entity is cached.

---

## @NaturalIdCache

Caches the `naturalId → id` mapping, enabling direct L2 lookup by business key:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@NaturalIdCache  // cache isbn→id mapping
public class Book {
    @NaturalId
    private String isbn;
}

// Direct L2 lookup — no SQL if both caches are warm
Book book = session.byNaturalId(Book.class)
    .using("isbn", "978-0-13-468599-1")
    .load();
```

---

## Query Cache

Caches the **IDs** of query results. Entities are still loaded from L2 (or DB if not cached).

```java
// Enable globally
spring.jpa.properties.hibernate.cache.use_query_cache=true

// Per query
@QueryHints(@QueryHint(name = HINT_CACHEABLE, value = "true"))
@Query("SELECT p FROM Post p WHERE p.status = :status ORDER BY p.createdAt DESC")
List<Post> findActiveOrderByDate(@Param("status") String status);
```

### Query Cache Invalidation Gotcha

**The query cache invalidates the ENTIRE query region when ANY entity of the queried type is inserted, updated, or deleted.**

```java
// This query is cached:
// SELECT id FROM post WHERE status = 'ACTIVE' → [1, 3, 7, 12]

// When ANY post is updated anywhere in the app:
// UPDATE post SET title = '...' WHERE id = 999;
// → entire query cache region for 'post' is INVALIDATED

// Next call: cache miss, re-runs the query
```

**Consequence:** Query cache is only beneficial for:
- Truly rarely-changing data (reference tables with query cache)
- Read-heavy applications with few writes
- Specific queries over largely static data

**Do NOT use query cache for:**
- Frequently updated entities
- Queries over large mutable datasets
- Paginated queries (too many cache entries)

---

## Provider Configuration

- **Ehcache** — most common; configure regions + TTLs in `ehcache.xml`. Remember to size `StandardQueryCache` separately.
- **Caffeine** — lightweight in-process, use `CaffeineDomainDataRegionFactory`.
- **Redis** — distributed; **use `READ_WRITE` or `TRANSACTIONAL`** — `NONSTRICT_READ_WRITE` has wider staleness windows across nodes.

---

## When Caching Hurts

| Scenario | Why It Hurts |
|---|---|
| Large entities with many fields | Memory pressure per cache entry |
| Entities updated by multiple processes (batch jobs + app server) | Cache staleness — processes bypass L2 |
| Native SQL updates bypassing Hibernate | Hibernate can't invalidate what it doesn't know about |
| Collections larger than 100 elements | Collection cache = entire collection in memory |

**Test your cache hit rate:**
```java
Statistics stats = sessionFactory.getStatistics();
double hitRatio = (double) stats.getSecondLevelCacheHitCount() /
    (stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount());
// Aim for > 0.9 (90%+ hits) — if lower, caching may not be worth the overhead
```
