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

---

## Query Plan Cache

Hibernate caches query execution plans. Avoid dynamic IN-list sizes:

```java
// ❌ BAD — new plan for each list size (1, 2, 3, ... unique cache entries)
List<Post> posts = postRepository.findAllById(Arrays.asList(1L, 2L, 3L));

// ✅ Better — use @BatchSize on the collection, or use HQL IN with fixed sizes
// Or configure: spring.jpa.properties.hibernate.query.plan_cache_max_size=2048
```

Tune the query plan cache:
```properties
spring.jpa.properties.hibernate.query.plan_cache_max_size=2048
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size=256
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
