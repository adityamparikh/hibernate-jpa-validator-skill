# Query Optimization

---

## DTO Projections — Load Only What You Need

Three flavors, all of which generate `SELECT <only-needed-columns>` instead of `SELECT *`:

- **Interface projection** — `interface PostSummary { Long getId(); String getTitle(); }` returned from a derived/`@Query` method. SpEL via `@Value("#{target.author.firstName + ' ' + target.author.lastName}")` for computed fields.
- **Record / class projection** — Java record + JPQL constructor expression: `SELECT new com.example.PostSummaryDTO(p.id, p.title, a.name) FROM Post p JOIN p.author a`.
- **Tuple projection** — `cb.createTupleQuery()` for fully dynamic Criteria selections.

---

## JPQL Best Practices

Always named parameters (`:status`) — never string concatenation. Reach for native SQL when JPQL can't express it: window functions (`ROW_NUMBER`, `RANK`, `LAG`), CTEs (`WITH`), database-specific functions, complex `GROUP BY` aggregations.

---

## Criteria API — Only for Dynamic Queries

Use Criteria only when predicates are conditional at runtime (search forms, filter builders). For any fixed query, JPQL is dramatically more readable. The pattern is a `List<Predicate>` built conditionally from non-null criteria fields, then `query.where(predicates.toArray(new Predicate[0]))`.

---

## Pagination: Keyset over OFFSET

### OFFSET Problem

```sql
-- OFFSET 10000, LIMIT 20
-- Database scans 10,020 rows, discards 10,000, returns 20
-- Gets slower with every page
SELECT * FROM post ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

### Keyset Pagination (Seek Method)

```java
// First page
List<Post> firstPage = postRepository
    .findTop20ByOrderByCreatedAtDesc();

// Next page — pass the last item's cursor values
@Query("SELECT p FROM Post p " +
       "WHERE p.createdAt < :lastCreatedAt OR " +
       "(p.createdAt = :lastCreatedAt AND p.id < :lastId) " +
       "ORDER BY p.createdAt DESC, p.id DESC")
List<Post> findNextPage(
    @Param("lastCreatedAt") Instant lastCreatedAt,
    @Param("lastId") Long lastId,
    Pageable pageable);
```

This does an **index seek** — O(log N) regardless of page depth.

### Hypersistence Utils: WindowIterator (clean keyset across all pages)

```java
import io.hypersistence.utils.spring.repository.WindowIterator;
import org.springframework.data.domain.ScrollPosition;
import org.springframework.data.domain.Window;

// Repository returns a Window<Post> for a given ScrollPosition
public interface PostRepository extends BaseJpaRepository<Post, Long> {
    Window<Post> findByStatus(String status, ScrollPosition position);
}

WindowIterator<Post> iterator = WindowIterator
    .of(ScrollPosition.keyset())                                    // start at beginning, keyset mode
    .limit(20)                                                      // page size
    .using(scrollPosition ->
        postRepository.findByStatus("ACTIVE", scrollPosition));     // your query

while (iterator.hasNext()) {
    Post p = iterator.next();
    process(p);
}
```

`WindowIterator` handles cursor advancement, page transitions, and termination — turns multi-page keyset traversal into a clean iterator. Ideal for batch jobs, exports, and any "process everything in chunks" pattern.

> Verify against the `hypersistence-utils` version on your classpath before copy-pasting; the API has evolved across releases.

### Spring Data 3.1+ ScrollPosition (built-in keyset)

```java
public interface PostRepository extends JpaRepository<Post, Long> {

    Window<Post> findFirst20ByStatusOrderByCreatedAtDesc(
        String status, ScrollPosition position);
}

// First page
Window<Post> page = postRepository.findFirst20ByStatusOrderByCreatedAtDesc(
    "ACTIVE", ScrollPosition.offset());

// Subsequent pages
while (page.hasNext()) {
    page = postRepository.findFirst20ByStatusOrderByCreatedAtDesc(
        "ACTIVE", page.positionAt(page.size() - 1));
}
```

`Window` is the keyset equivalent of `Page` — returns `hasNext()` without a COUNT query.

---

## Query Plan Cache

Hibernate caches the HQL/JPQL parse tree and translation for each unique query string. Every distinct query string gets its own cache entry.

### The IN-List Explosion Problem

```java
// Each of these generates a DIFFERENT query plan:
// "... WHERE id IN (?)"
// "... WHERE id IN (?,?)"
// "... WHERE id IN (?,?,?)"
// For an app with lists of 1–1000 items: 1000 distinct cache entries → GC pressure

List<Post> posts = postRepository.findAllById(Arrays.asList(1L, 2L, 3L));
```

### Fix 1: in_clause_parameter_padding (Recommended)

```properties
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
```

This rounds up IN-list sizes to the next power of 2: lists of 1–2 items → 2 params, 3–4 → 4, 5–8 → 8, etc. Dramatically reduces unique plan count (log₂ N distinct sizes instead of N).

```sql
-- Without padding: IN (?,?,?)
-- With padding:    IN (?,?,?,?)  ← padded to 4, last param repeated
-- Same query plan reused for all lists of size 3–4
```

**Trade-off:** Slightly larger IN clause (never more than 2x the actual list size).

### Fix 2: Tune Cache Size

```properties
spring.jpa.properties.hibernate.query.plan_cache_max_size=2048         # default 2048
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size=128  # default 128
```

Increase `plan_cache_max_size` if your app has many distinct queries and you see cache eviction in statistics (`hibernate.query.plan_cache_miss_count` rising).

### Fix 3: Use @BatchSize for Collection Fetching

`@BatchSize` generates fixed-size IN clauses rather than variable-size ones:
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=32  # always uses 32-slot IN clauses
```

---

## @QueryHints

```java
// Read-only hint — skips dirty checking, saves memory
@QueryHints(@QueryHint(name = HINT_READONLY, value = "true"))
@Query("SELECT p FROM Post p WHERE p.status = :status")
List<Post> findReadOnly(@Param("status") String status);

// Fetch size — stream large result sets efficiently
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "50"))
@Query("SELECT p FROM Post p")
Stream<Post> streamAll();
```

---

## JOIN FETCH + Pagination — the HHH-1010 Trap

```java
// ❌ This combination forces in-memory pagination
@Query("SELECT p FROM Post p JOIN FETCH p.comments")
Page<Post> findAllWithComments(Pageable pageable);

// Hibernate logs: "HHH90003004: firstResult/maxResults specified with collection
//                  fetch; applying in memory!"
// Translation: ALL rows loaded, then trimmed in Java. Catastrophic on large tables.
```

**Why:** Joining a collection produces a Cartesian product. Applying `LIMIT/OFFSET` to that would slice mid-way through a parent's children, returning broken graphs. Hibernate's only safe option is to fetch everything and slice in memory.

### Fix 1: Two-Query Pattern (recommended)

```java
// Step 1 — paginate parents, no fetch
@Query("SELECT p.id FROM Post p WHERE p.status = :status ORDER BY p.id")
List<Long> findIds(@Param("status") String status, Pageable pageable);

// Step 2 — load with JOIN FETCH, no pagination
@Query("SELECT DISTINCT p FROM Post p JOIN FETCH p.comments WHERE p.id IN :ids")
List<Post> findWithComments(@Param("ids") List<Long> ids);

// Service composes the two:
public List<Post> page(int pageNum, int size) {
    List<Long> ids = repo.findIds("ACTIVE", PageRequest.of(pageNum, size));
    return ids.isEmpty() ? List.of() : repo.findWithComments(ids);
}
```

This gives you correct pagination AND the fetched collection — two queries, no in-memory slicing.

### Fix 2: @BatchSize on the Collection

```java
@OneToMany(mappedBy = "post")
@BatchSize(size = 25)
private List<Comment> comments;

// Now you can paginate naturally — children load in batches
@Query("SELECT p FROM Post p WHERE p.status = :status")
Page<Post> findActive(@Param("status") String status, Pageable pageable);
```

Adds one extra query per page (the batched comment fetch), but allows native pagination.

---

## Emulating LEFT JOIN FETCH via Projections

When you only need a small slice of an associated entity, an interface projection generates a leaner SQL than `JOIN FETCH`:

```java
public interface PostWithAuthorView {
    Long getId();
    String getTitle();
    AuthorView getAuthor();   // nested projection — NOT a full Author entity

    interface AuthorView {
        Long getId();
        String getName();
    }
}

public interface PostRepository extends JpaRepository<Post, Long> {
    List<PostWithAuthorView> findByStatus(String status);
}
```

Generated SQL:
```sql
SELECT p.id, p.title, a.id, a.name
FROM post p
LEFT JOIN author a ON p.author_id = a.id
WHERE p.status = ?
```

vs. JOIN FETCH which loads `SELECT p.*, a.*` — every column of every entity.

**When to use which:**
- **JOIN FETCH** — write paths, full entity needed, will be modified
- **Projection with nested view** — read paths, render-only, large entity tables
- **Blaze Persistence Entity Views** — when you need projections + collections without N+1 (it generates window functions to avoid the Cartesian)

---

## Native SQL for Reporting

Use `@Query(nativeQuery = true)` for window functions, CTEs, and complex `GROUP BY` aggregations. For type safety, map results into DTOs via `@SqlResultSetMapping` + `@ConstructorResult` instead of `List<Object[]>`.

---

## Blaze Persistence (for complex Criteria)

When Criteria API becomes unwieldy, Blaze Persistence provides a fluent builder over JPQL with CTE support:

```java
CriteriaBuilder<Post> cb = criteriaBuilderFactory.create(entityManager, Post.class);
cb.where("author.name").like().value("%" + authorName + "%").noEscape()
  .where("status").eq("ACTIVE")
  .orderByDesc("createdAt");
List<Post> posts = cb.getResultList();
```

Supports: CTEs, window functions, keyset pagination, UPDATE/DELETE with joins.

---

## jOOQ Alongside Hibernate

For complex reporting, use jOOQ for queries and Hibernate for writes:

```java
// jOOQ for reads — type-safe, full SQL power
Result<Record3<String, Integer, LocalDateTime>> result = dsl
    .select(AUTHOR.NAME, count(POST.ID), max(POST.CREATED_AT))
    .from(AUTHOR)
    .leftJoin(POST).on(POST.AUTHOR_ID.eq(AUTHOR.ID))
    .groupBy(AUTHOR.ID, AUTHOR.NAME)
    .fetch();

// Hibernate for writes — entity lifecycle management
Post post = new Post("New title");
post.setAuthor(entityManager.getReference(Author.class, authorId));
entityManager.persist(post);
```
