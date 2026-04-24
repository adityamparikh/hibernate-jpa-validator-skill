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
