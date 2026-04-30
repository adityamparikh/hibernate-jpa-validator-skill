# Query Optimization

---

## DTO Projections — Load Only What You Need

Loading full entities for read-only display is the most common performance waste in JPA applications.

### Interface Projections (simplest)

```java
// Define the projection
public interface PostSummary {
    Long getId();
    String getTitle();
    @Value("#{target.author.firstName + ' ' + target.author.lastName}")
    String getAuthorFullName();  // SpEL — computed at runtime
}

// Repository method — Hibernate generates optimal SELECT
public interface PostRepository extends JpaRepository<Post, Long> {
    List<PostSummary> findByStatus(String status);
}
```

Generated SQL:
```sql
SELECT p.id, p.title, a.first_name, a.last_name
FROM post p
JOIN author a ON p.author_id = a.id
WHERE p.status = ?
```

Not `SELECT *`.

### Class/Record Projections (better for complex cases)

```java
// Java record (preferred in modern Java)
public record PostSummaryDTO(Long id, String title, String authorName) {}

// JPQL constructor expression
@Query("SELECT new com.example.dto.PostSummaryDTO(p.id, p.title, a.name) " +
       "FROM Post p JOIN p.author a WHERE p.status = :status")
List<PostSummaryDTO> findSummaries(@Param("status") String status);
```

### Tuple Projections (dynamic queries)

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Tuple> query = cb.createTupleQuery();
Root<Post> post = query.from(Post.class);
query.multiselect(
    post.get("id").alias("id"),
    post.get("title").alias("title")
);
List<Tuple> results = entityManager.createQuery(query).getResultList();
```

---

## JPQL Best Practices

```java
// ✅ Named parameter (safe, cacheable)
@Query("SELECT p FROM Post p WHERE p.author.id = :authorId AND p.status = :status")
List<Post> findByAuthorAndStatus(@Param("authorId") Long authorId,
                                  @Param("status") String status);

// ❌ String concatenation — SQL injection risk and no query plan cache reuse
String jpql = "SELECT p FROM Post p WHERE p.status = '" + status + "'";
```

**JPQL limitations → use native SQL:**
- Window functions (ROW_NUMBER, RANK, LAG)
- CTEs (WITH clauses)
- Database-specific functions
- Reporting aggregations with complex GROUP BY

---

## Criteria API — Only for Dynamic Queries

```java
// ✅ Criteria for truly dynamic query building
public List<Post> search(PostSearchCriteria criteria) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<Post> query = cb.createQuery(Post.class);
    Root<Post> post = query.from(Post.class);
    List<Predicate> predicates = new ArrayList<>();

    if (criteria.getStatus() != null) {
        predicates.add(cb.equal(post.get("status"), criteria.getStatus()));
    }
    if (criteria.getAuthorId() != null) {
        predicates.add(cb.equal(post.get("author").get("id"), criteria.getAuthorId()));
    }
    if (criteria.getFromDate() != null) {
        predicates.add(cb.greaterThanOrEqualTo(post.get("createdAt"), criteria.getFromDate()));
    }

    query.where(predicates.toArray(new Predicate[0]));
    return entityManager.createQuery(query).getResultList();
}

// ❌ Don't use Criteria for fixed queries — JPQL is far more readable
```

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

```java
// Complex reporting query — don't try to express in JPQL
@Query(
    value = """
        SELECT
            a.name AS author_name,
            COUNT(p.id) AS post_count,
            AVG(LENGTH(p.body)) AS avg_body_length,
            MAX(p.created_at) AS last_post_date
        FROM author a
        LEFT JOIN post p ON p.author_id = a.id
        WHERE a.active = true
        GROUP BY a.id, a.name
        HAVING COUNT(p.id) > :minPosts
        ORDER BY post_count DESC
        """,
    nativeQuery = true
)
List<Object[]> getAuthorStats(@Param("minPosts") int minPosts);

// Prefer SqlResultSetMapping or native query to DTO for type safety:
@SqlResultSetMapping(
    name = "AuthorStatMapping",
    classes = @ConstructorResult(
        targetClass = AuthorStatDTO.class,
        columns = {
            @ColumnResult(name = "author_name", type = String.class),
            @ColumnResult(name = "post_count", type = Long.class)
        }
    )
)
```

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
