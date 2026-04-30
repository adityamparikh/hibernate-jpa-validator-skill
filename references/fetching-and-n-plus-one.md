# Fetching and N+1 Detection

---

## How N+1 Happens

```java
// This service method generates N+1:
public List<PostDTO> getAllPosts() {
    List<Post> posts = postRepository.findAll();  // SELECT * FROM post (1 query)
    return posts.stream()
        .map(p -> new PostDTO(
            p.getId(),
            p.getTitle(),
            p.getAuthor().getName()  // SELECT * FROM author WHERE id=? (N queries!)
        ))
        .toList();
}
```

For 100 posts: 1 + 100 = 101 SQL queries. Hibernate logs each as DEBUG — users experience it as slow page loads.

The root cause is always the same: **lazy association accessed in a loop without a pre-fetch plan**.

---

## Detecting N+1

### Option 1: Hibernate Statistics (built-in, zero deps)

```properties
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
logging.level.org.hibernate.SQL=DEBUG
```

Log output will show:
```
Session Metrics {
  ...
  24 JDBC statements executed (expected 1)
  ...
}
```

### Option 2: datasource-proxy (recommended for tests)

```java
// In test config:
@Bean
DataSource dataSource() {
    return ProxyDataSourceBuilder
        .create(realDataSource())
        .name("DS-Proxy")
        .logQueryBySlf4j()
        .countQuery()          // count queries per test
        .build();
}

// Assert exact query count in test:
assertThat(queryCountHolder.getTotal()).isEqualTo(1);
```

### Option 3: p6spy (prod-safe logging)

Add `spy.properties` and `p6spy` dependency. Logs every SQL with timing.

---

## Fix Strategies

### 1. JOIN FETCH (explicit JPQL)

Best for: a single query where you always need the association.

```java
// JPQL
@Query("SELECT p FROM Post p JOIN FETCH p.author WHERE p.status = :status")
List<Post> findActiveWithAuthor(@Param("status") String status);

// With multiple levels
@Query("SELECT p FROM Post p JOIN FETCH p.author a JOIN FETCH a.address WHERE p.id = :id")
Optional<Post> findWithFullGraph(@Param("id") Long id);
```

**Limitation:** Cannot `JOIN FETCH` two collection associations in one query → `MultipleBagFetchException` (or Cartesian product with Sets).

### 2. @EntityGraph (declarative, reusable)

Best for: Spring Data repository methods where you want different fetch plans per call.

```java
// Define on entity
@Entity
@NamedEntityGraph(
    name = "Post.withAuthorAndTags",
    attributeNodes = {
        @NamedAttributeNode("author"),
        @NamedAttributeNode("tags")
    }
)
public class Post { ... }

// Use in repository
@EntityGraph("Post.withAuthorAndTags")
Optional<Post> findById(Long id);

// Or inline (Hibernate 6+ style)
@EntityGraph(attributePaths = {"author", "tags"})
List<Post> findByStatus(String status);
```

**Note:** `@EntityGraph` generates a LEFT JOIN FETCH — returns duplicate roots when fetching collections. Use `DISTINCT` or a `Set` return type, or use `@QueryHints`.

### 3. @BatchSize (N queries → ceil(N/batch) queries)

Best for: collections where you're loading multiple parent entities and need their children.

```java
@BatchSize(size = 25)
@OneToMany(mappedBy = "post")
private List<Comment> comments;
```

For 100 posts: instead of 100 comment queries, generates:
```sql
SELECT * FROM comment WHERE post_id IN (1,2,...,25);   -- batch 1
SELECT * FROM comment WHERE post_id IN (26,27,...,50);  -- batch 2
SELECT * FROM comment WHERE post_id IN (51,...,75);     -- batch 3
SELECT * FROM comment WHERE post_id IN (76,...,100);    -- batch 4
```

Can also be set globally:
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### 4. @Fetch(FetchMode.SUBSELECT)

Best for: loading children for ALL loaded parents in one query.

```java
@Fetch(FetchMode.SUBSELECT)
@OneToMany(mappedBy = "post")
private List<Comment> comments;
```

Generates:
```sql
SELECT * FROM comment WHERE post_id IN (
    SELECT id FROM post WHERE status = 'ACTIVE'  -- reuses original query
);
```

**Use SUBSELECT when:** You always need all children for all parents — saves the IN-list construction overhead of BatchSize.

---

## MultipleBagFetchException

```java
// ❌ This query throws MultipleBagFetchException
@Query("SELECT p FROM Post p JOIN FETCH p.comments JOIN FETCH p.tags")
List<Post> findAllWithCommentsAndTags();
```

Hibernate refuses to JOIN FETCH two `List` (bag) collections simultaneously because it cannot de-duplicate the Cartesian product correctly.

**Fixes:**

Option A — Change to `Set`:
```java
private Set<Comment> comments = new LinkedHashSet<>();
private Set<Tag> tags = new LinkedHashSet<>();
```
Now both can be JOIN FETCHed. Still produces a Cartesian product in SQL but Hibernate de-dupes correctly with Sets.

Option B — Fetch in separate queries:
```java
// Query 1: load posts with comments
@Query("SELECT DISTINCT p FROM Post p JOIN FETCH p.comments WHERE p.id IN :ids")
List<Post> findWithComments(@Param("ids") List<Long> ids);

// Query 2: load tags for same posts (Hibernate merges into first-level cache)
@Query("SELECT DISTINCT p FROM Post p JOIN FETCH p.tags WHERE p.id IN :ids")
List<Post> findWithTags(@Param("ids") List<Long> ids);
```

Option C — Use `@BatchSize` on both collections (cleanest for most cases).

---

## When NOT to Use JOIN FETCH

- When paginating with `LIMIT/OFFSET` — Hibernate will warn: "HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!" This means ALL rows are loaded, then paginated in Java. Use `@BatchSize` or a subquery approach instead.
- When the association is not needed in this specific call path — just leave it lazy

---

## Persistence Context Size Management

The first-level cache (persistence context) holds every entity loaded or persisted in the current Session. It's a `HashMap<EntityKey, Object>`. Unbounded growth causes:
- **Memory pressure** — hundreds of thousands of objects pinned in heap
- **Slow flush** — dirty checking compares every managed entity
- **Stale data** — reading the same entity twice returns the cached version even if DB changed

### Signs of an Oversized Persistence Context

```java
// ❌ ANTI-PATTERN: loads entire table into persistence context
public void updateAllPrices() {
    List<Product> all = productRepository.findAll();  // 500,000 products in L1 cache!
    all.forEach(p -> p.setPrice(p.getPrice().multiply(BigDecimal.valueOf(1.10))));
}

// Flush: Hibernate dirty-checks all 500,000 entities × all fields = very slow
```

### Rule: Keep Transactions Short and Narrow

```java
// ✅ Short transaction — only what's needed
@Transactional
public void adjustPrice(Long productId, BigDecimal multiplier) {
    Product p = productRepository.findById(productId).orElseThrow();
    p.setPrice(p.getPrice().multiply(multiplier));
    // Transaction commits, L1 cache discarded — no accumulation
}
```

### Rule: Use JPQL Bulk UPDATE Instead of Load-and-Modify

```java
// ❌ 500,000 entities in persistence context, 500,000 UPDATEs
List<Product> products = productRepository.findByCategory("ELECTRONICS");
products.forEach(p -> p.setPrice(p.getPrice().multiply(BigDecimal.valueOf(1.10))));

// ✅ Zero entities in persistence context, 1 UPDATE
@Modifying(clearAutomatically = true)
@Query("UPDATE Product p SET p.price = p.price * :multiplier WHERE p.category = :category")
int adjustPrices(@Param("multiplier") BigDecimal multiplier, @Param("category") String category);
```

### Rule: Detach Read-Only Entities After Use

```java
@Transactional(readOnly = true)
public List<PostDTO> buildReport() {
    List<Post> posts = postRepository.findAll();

    List<PostDTO> result = posts.stream()
        .map(postMapper::toDTO)
        .toList();

    // Explicitly evict from L1 to free memory — optional in readOnly=true tx
    // but good practice in long-lived sessions
    posts.forEach(entityManager::detach);

    return result;
}
```

`@Transactional(readOnly = true)` hints to Hibernate to skip the snapshot copy for dirty checking. It does **not** prevent the entities from being held in the L1 cache for the transaction duration — `detach()` or `clear()` does.

### Rule: Never Load More Than ~1000 Entities per Transaction

If you need to process more, use:
1. Chunk processing with flush/clear (see batch-processing.md)
2. `ScrollableResults` with flush/clear (see batch-processing.md)
3. `StatelessSession` for pure bulk operations

---

## Fetch Decision Tree

```
Need the association in this call?
├── No → Leave LAZY, don't fetch
├── Yes, one specific entity → JOIN FETCH in @Query
├── Yes, one entity type, multiple instances → @BatchSize(25) on mapping
├── Yes, all parents need all children → @Fetch(SUBSELECT)
└── Yes, complex graph, Spring Data → @EntityGraph
```
