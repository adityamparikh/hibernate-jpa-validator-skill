# Spring Data JPA Patterns

---

## Repository Hierarchy

```
Repository (marker)
└── CrudRepository<T, ID>
    └── PagingAndSortingRepository<T, ID>
        └── JpaRepository<T, ID>
            └── Your PostRepository
```

Extend `JpaRepository` unless you specifically want to hide methods (then extend `CrudRepository` or `Repository`).

### The JpaRepository Trap: `findAll()` and `findById()`

`JpaRepository` ships with two methods that are almost always the wrong choice:

```java
// ❌ findAll() — loads the entire table into memory + L1 cache
List<Post> all = postRepository.findAll();

// ❌ findById() — calls em.find(), which fetches all EAGER associations,
//    runs every @PostLoad listener, and loads into the persistence context
Optional<Post> p = postRepository.findById(id);
```

Vlad's specific guidance:
- **`findAll()`** — never paginate by loading everything; never use as a filter base. Use `findByXxx`, `Pageable`, or a projection.
- **`findById()`** — for write paths where you already know the entity exists, prefer `getReferenceById()` (returns a proxy, no SELECT until you access state):

```java
// You only need a managed reference to set as a foreign key:
Post post = new Post();
post.setAuthor(authorRepository.getReferenceById(authorId));  // no SELECT
postRepository.save(post);                                     // single INSERT
```

### BaseJpaRepository (Hypersistence Utils)

Vlad's `BaseJpaRepository` deliberately omits the dangerous methods and adds the ones you actually want:

```xml
<dependency>
    <groupId>io.hypersistence</groupId>
    <artifactId>hypersistence-utils-hibernate-63</artifactId>
</dependency>
```

```java
public interface PostRepository extends BaseJpaRepository<Post, Long> { }

// Available methods:
postRepository.persist(post);             // explicit INSERT (no merge fallback)
postRepository.merge(post);               // explicit MERGE (no automatic decision)
postRepository.findById(id);              // returns Optional<Post>
postRepository.getReferenceById(id);      // proxy — no SELECT
postRepository.persistAndFlush(post);
postRepository.lockById(id, LockMode.PESSIMISTIC_WRITE);
// NO findAll(), NO save() — gone by design
```

The key insight: `save()` is ambiguous (insert or merge?). Splitting into `persist`/`merge` makes intent explicit and avoids the unnecessary SELECT that `merge` triggers for new entities.

### `save()` vs `persist()`

`SimpleJpaRepository.save()` checks `entityInformation.isNew(entity)`:
- **Generated ID + null id field** → `isNew == true` → calls `em.persist()` (single INSERT, no extra SELECT). Behaviour is fine.
- **Assigned ID** (`@Id` set manually, no `@GeneratedValue`) → `isNew == false` → calls `em.merge()`, which **does** issue a SELECT before INSERT and returns a *different* managed instance. The original is left detached.
- **`Persistable<ID>` override** → whatever `isNew()` you implemented decides.

```java
// ⚠️ With assigned IDs (e.g. UUID set in the constructor), save() goes through MERGE:
Post incoming = new Post(UUID.randomUUID(), "Hello");
Post managed = postRepository.save(incoming);   // SELECT then INSERT
// `incoming` is detached; mutating `incoming` after this point won't be flushed.
// Always use the returned reference (`managed`).

// ✅ persist() from BaseJpaRepository makes intent explicit and skips the MERGE branch.
postRepository.persist(post);
// `post` is now managed — getId() reflects the generated key, no extra SELECT.
```

Even when `save()` happens to do the right thing for `@GeneratedValue`, `persist()` documents intent (insert, never merge) and prevents accidents when an entity is later switched to assigned IDs.

---

## Derived Query Methods

```java
// ✅ Simple cases
List<Post> findByStatus(String status);
Optional<Post> findBySlug(String slug);
boolean existsBySlug(String slug);
long countByStatus(String status);

// ✅ With modifiers
List<Post> findTop10ByStatusOrderByCreatedAtDesc(String status);
Page<Post> findByStatus(String status, Pageable pageable);
Slice<Post> findByStatus(String status, Pageable pageable);  // no COUNT query

// ❌ Don't let derived methods generate inefficient SQL
List<Post> findByAuthorNameContaining(String name);
// Generates: WHERE a.name LIKE '%name%' — full table scan, no index
// Use @Query with native FULLTEXT search instead
```

**Slice vs Page:**
- `Page` runs a COUNT query to compute total pages (slow on large tables)
- `Slice` only checks if there's a next page (one extra row) — use when you don't need total count

---

## @Query with JPQL

```java
// Named parameters (preferred)
@Query("SELECT p FROM Post p WHERE p.author.id = :authorId AND p.createdAt >= :from")
List<Post> findByAuthorSince(@Param("authorId") Long authorId, @Param("from") Instant from);

// DTO projection with constructor expression
@Query("SELECT new com.example.dto.PostSummary(p.id, p.title, a.name) " +
       "FROM Post p JOIN p.author a WHERE p.status = 'ACTIVE'")
List<PostSummary> findActiveSummaries();

// Avoid SELECT * when only a few fields needed
@Query("SELECT p.id, p.title, p.slug FROM Post p WHERE p.status = :status")
List<Object[]> findIdTitleSlug(@Param("status") String status);  // or interface projection
```

---

## EXISTS Query Optimization

```java
// ❌ WRONG — counts every matching row, then compares to 0
boolean hasPost = postRepository.countBySlug(slug) > 0;
// SQL: SELECT COUNT(*) FROM post WHERE slug = ?

// ❌ Also wrong — same shape as the COUNT-based check
@Query("SELECT CASE WHEN COUNT(p) > 0 THEN true ELSE false END FROM Post p WHERE p.slug = :slug")
boolean existsBySlugViaCount(@Param("slug") String slug);

// ✅ Spring Data 3.x existsByXxx — generates SELECT 1 ... LIMIT 1
//    The optimizer stops scanning at the first matching row. This is the right default.
boolean exists = postRepository.existsBySlug(slug);
// SQL: SELECT 1 FROM post WHERE slug = ? FETCH FIRST 1 ROWS ONLY

// ✅ Native EXISTS — equivalent and explicit; useful when EXISTS is part of a larger query
@Query(value = "SELECT EXISTS(SELECT 1 FROM post WHERE slug = :slug)", nativeQuery = true)
boolean existsBySlugNative(@Param("slug") String slug);
```

Reach for the `existsByXxx` derived method by default. The COUNT form is what you should be replacing; the native EXISTS is a tool for composing into bigger queries, not a meaningful speed-up over the derived method on a hot path.

> Older Spring Data Commons (≤ 2.x) generated `SELECT 1 FROM ...` without `LIMIT 1` for some store implementations — if you're on an old stack, verify the SQL with `datasource-proxy`.

---

## Stream Methods

```java
// Returns a Stream — must be consumed inside @Transactional
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "50"))
@Query("SELECT p FROM Post p WHERE p.status = :status")
Stream<Post> streamByStatus(@Param("status") String status);

@Transactional(readOnly = true)
public void exportActivePosts(Writer writer) {
    try (Stream<Post> stream = postRepository.streamByStatus("ACTIVE")) {
        stream.forEach(p -> {
            writer.write(toCsv(p));
            entityManager.detach(p);  // ← critical: don't accumulate in L1 cache
        });
    }
}
```

**Three rules:**
1. Always close the stream (try-with-resources).
2. Always detach each entity after processing — otherwise you leak memory at scale.
3. Always inside `@Transactional` — the underlying ResultSet must stay open.

For pure bulk processing (no entity lifecycle), prefer `StatelessSession` — see `references/batch-processing.md`.

---

## Query By Example (QBE)

```java
public interface PostRepository extends JpaRepository<Post, Long>,
                                        QueryByExampleExecutor<Post> { }

// Build a probe entity — non-null fields become equality predicates
Post probe = new Post();
probe.setStatus("ACTIVE");
probe.setAuthor(authorRepository.getReferenceById(authorId));

ExampleMatcher matcher = ExampleMatcher.matchingAll()
    .withMatcher("title", contains().ignoreCase())
    .withIgnorePaths("createdAt", "updatedAt");

List<Post> results = postRepository.findAll(Example.of(probe, matcher));
```

**Limitations** (why you usually want `Specification` instead):
- No `OR` predicates — only `AND`
- No range predicates (`<`, `>`, `BETWEEN`)
- No `IN` lists or nested associations beyond `equal`
- Probe must be a fully constructed entity — awkward for required fields

Use QBE only for simple "match these fields" search forms. For anything richer, use `Specification` or Blaze Persistence.

---

## @Modifying — clearAutomatically is Required

```java
// ❌ WRONG — stale first-level cache after bulk update
@Modifying
@Query("UPDATE Post p SET p.status = :status WHERE p.author.id = :authorId")
int updateStatusByAuthor(@Param("status") String status, @Param("authorId") Long authorId);

// ✅ CORRECT
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Post p SET p.status = :status WHERE p.author.id = :authorId")
int updateStatusByAuthor(@Param("status") String status, @Param("authorId") Long authorId);
```

`flushAutomatically = true`: flush pending changes before executing the DML (so you don't overwrite unsaved changes).
`clearAutomatically = true`: clear first-level cache after DML (so subsequent reads reflect the update).

---

## Projections

### Interface Projection (simplest, lazy)

```java
public interface PostListView {
    Long getId();
    String getTitle();
    String getSlug();

    // Nested projection — avoids loading full Author entity
    AuthorView getAuthor();

    interface AuthorView {
        String getName();
    }
}

List<PostListView> findByStatus(String status);
// SQL: SELECT p.id, p.title, p.slug, a.name FROM post p JOIN author a ...
```

### Record Projection (Java 16+, recommended for DTOs)

```java
public record PostRecord(Long id, String title, String slug) {}

@Query("SELECT new com.example.dto.PostRecord(p.id, p.title, p.slug) FROM Post p WHERE p.status = :status")
List<PostRecord> findAsRecords(@Param("status") String status);
```

---

## Custom Repository Fragments

For complex queries that don't fit in `@Query`:

```java
// 1. Define fragment interface
public interface PostRepositoryCustom {
    List<Post> searchWithFilters(PostSearchCriteria criteria);
}

// 2. Implement it
public class PostRepositoryCustomImpl implements PostRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Post> searchWithFilters(PostSearchCriteria criteria) {
        // CriteriaBuilder / JPQL / native SQL
    }
}

// 3. Compose
public interface PostRepository extends JpaRepository<Post, Long>, PostRepositoryCustom { }
```

---

## Specification API (Dynamic Filters)

```java
public class PostSpecifications {

    public static Specification<Post> hasStatus(String status) {
        return (root, query, cb) ->
            status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
    }

    public static Specification<Post> createdAfter(Instant date) {
        return (root, query, cb) ->
            date == null ? cb.conjunction() : cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }

    public static Specification<Post> authorNameContains(String name) {
        return (root, query, cb) -> {
            if (name == null) return cb.conjunction();
            Join<Post, Author> author = root.join("author", JoinType.LEFT);
            return cb.like(cb.lower(author.get("name")), "%" + name.toLowerCase() + "%");
        };
    }
}

// Usage
Page<Post> results = postRepository.findAll(
    where(hasStatus("ACTIVE"))
        .and(createdAfter(from))
        .and(authorNameContains(nameFilter)),
    PageRequest.of(0, 20)
);
```

---

## Auditing

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig { }

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(nullable = false, updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(nullable = false, length = 100)
    private String lastModifiedBy;
}

@Component
public class AuditorAwareImpl implements AuditorAware<String> {
    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext())
            .map(ctx -> ctx.getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

---

## Anti-Patterns

### findAll() + Java Filter

```java
// ❌ WRONG — loads entire table, filters in Java
List<Post> active = postRepository.findAll().stream()
    .filter(p -> "ACTIVE".equals(p.getStatus()))
    .toList();

// ✅ CORRECT — filter in SQL
List<Post> active = postRepository.findByStatus("ACTIVE");
```

### save() on Managed Entity

```java
// ❌ Redundant — entity is already managed, dirty checking handles it
@Transactional
public void updateTitle(Long id, String newTitle) {
    Post post = postRepository.findById(id).orElseThrow();
    post.setTitle(newTitle);
    postRepository.save(post);  // unnecessary — Hibernate will UPDATE at flush time
}

// ✅ Correct — just modify, let dirty checking work
@Transactional
public void updateTitle(Long id, String newTitle) {
    Post post = postRepository.findById(id).orElseThrow();
    post.setTitle(newTitle);
    // no save() needed
}
```

### N+1 via Derived Query

```java
// ❌ Derived query triggers N+1 for author
List<Post> posts = postRepository.findByStatus("ACTIVE");
posts.forEach(p -> render(p.getAuthor().getName()));  // N queries for authors

// ✅ Fetch with JOIN
@EntityGraph(attributePaths = "author")
List<Post> findByStatus(String status);
```

### Returning Entities from REST Controllers

```java
// ❌ Lazy associations + open-in-view = silent N+1 + tight coupling to schema
@GetMapping("/posts")
public List<Post> listPosts() {
    return postRepository.findAll();  // serializer triggers lazy loads
}

// ✅ DTO projection in service, controller returns DTOs only
@GetMapping("/posts")
public List<PostSummaryDTO> listPosts() {
    return postService.findActiveSummaries();
}
```

Always return DTOs from controllers. This:
- Surfaces lazy-loading bugs as exceptions (with `open-in-view=false`) instead of N+1
- Decouples API contract from entity schema
- Eliminates accidental write-through (Jackson can't serialize-then-mutate a managed entity if it never sees one)

---

## Bidirectional Association Sync Helpers

When using `@OneToMany(mappedBy = ...)`, the **child** owns the FK. Setting the parent on the child is what persists the relationship:

```java
@Entity
public class Post {
    @OneToMany(mappedBy = "post", cascade = ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    // ✅ Always pair add/remove — keeps in-memory state consistent with DB
    public void addComment(Comment c) {
        comments.add(c);
        c.setPost(this);
    }

    public void removeComment(Comment c) {
        comments.remove(c);
        c.setPost(null);  // triggers orphanRemoval — DELETE FROM comment WHERE id=?
    }
}

@Entity
public class Comment {
    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;
}
```

**Why both sides matter:**
- Without `c.setPost(this)`: the FK is null on flush → constraint violation or orphan
- Without `comments.add(c)`: in-memory state diverges; iterating `post.getComments()` won't see the new comment until reload

For `equals`/`hashCode` in this scenario, see `references/entity-mapping-checklist.md` — child entities used in a parent's `Set` must have a stable hash.

---

## Jakarta Data (Spring Boot 3.4+ / Jakarta EE 11)

Jakarta Data is the standardized successor to Spring Data — same derived-query pattern, but spec-defined.

```java
@Repository
public interface PostRepository extends BasicRepository<Post, Long> {

    @Find
    List<Post> findByStatus(String status);

    @Query("SELECT p FROM Post p WHERE p.author.id = :authorId")
    Page<Post> byAuthor(Long authorId, PageRequest page);

    @Insert
    Post create(Post post);

    @Update
    Post update(Post post);

    @Delete
    void delete(Post post);
}
```

Key differences from Spring Data:
- Explicit `@Insert` / `@Update` / `@Delete` (no ambiguous `save()`)
- Built-in keyset pagination via `CursoredPage`
- `@Query` without provider-specific extensions — portable across Hibernate, EclipseLink, etc.

Spring Boot 3.4+ supports both Jakarta Data interfaces and Spring Data interfaces side-by-side. New code can pick either.

---

## OSIV (Open Session In View) — Turn It Off

```properties
spring.jpa.open-in-view=false
```

Default is `true`, which keeps the Hibernate session open for the entire HTTP request. This silently masks N+1 in controllers and view layers, and holds DB connections longer than needed.

With `open-in-view=false`:
- All entity loading must happen inside `@Transactional` service methods
- Lazy access from controllers throws `LazyInitializationException` — loud and immediate
- Forces DTO projection — which is the right pattern anyway

→ See `references/spring-transactions.md` for full transaction architecture.
