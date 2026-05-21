# Spring Data JPA Patterns

---

## Repository Hierarchy

`Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository`. Extend `JpaRepository` unless you specifically want to hide methods.

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

Standard derived shapes: `findByXxx`, `existsByXxx`, `countByXxx`, `findTop10ByXxxOrderByYyyDesc`, with optional `Pageable`/`Sort`/`Limit` params.

**Slice vs Page:** `Page` runs an extra `COUNT(*)` to compute `getTotalPages()` (slow on large tables). `Slice` just fetches `pageSize + 1` rows and reports `hasNext()`. Use `Slice` whenever the UI doesn't need a total.

**Watch out:** `findByAuthorNameContaining(name)` generates `WHERE a.name LIKE '%name%'` — full table scan, no index. Use a `@Query` with native fulltext (PostgreSQL `tsvector`, MySQL `MATCH AGAINST`) instead.

---

## @Query with JPQL

Named parameters (`:authorId`) only — never string concatenation. For projections, use JPQL constructor expressions (`SELECT new com.example.PostSummary(p.id, p.title, a.name) FROM Post p JOIN p.author a`) or interface projections rather than `List<Object[]>`.

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

Extend `QueryByExampleExecutor<Post>`, build a probe entity with non-null fields → equality predicates, run `findAll(Example.of(probe, matcher))`.

**Severely limited:** AND only (no OR), no range/`IN`/nested associations beyond equality. Use only for trivial "match these fields" search forms; reach for `Specification` for anything else.

---

## @Modifying — clearAutomatically is Required

`@Modifying(clearAutomatically = true, flushAutomatically = true)` is mandatory on bulk DML. Without `flushAutomatically`, pending L1 changes get clobbered by the bulk UPDATE; without `clearAutomatically`, subsequent reads return pre-update cached entities.

---

## Projections

See `references/query-optimization.md` for the projection mechanics. Spring Data-specific notes: interface projections support **nested** views (e.g., a `PostListView` with an `AuthorView getAuthor()` returning only `id`/`name` — generates a JOIN with only those columns, never loads the full `Author` entity).

---

## Custom Repository Fragments

For queries that don't fit in `@Query`: define a `PostRepositoryCustom` interface, implement it as `PostRepositoryCustomImpl` (Spring matches by name + `Impl`) with `@PersistenceContext EntityManager`, then `PostRepository extends JpaRepository<...>, PostRepositoryCustom`.

---

## Specification API (Dynamic Filters)

`Specification<Post>` is a `(root, query, cb) -> Predicate` lambda — compose with `where(specA).and(specB).or(specC)`. Pattern: each spec checks its input for null and returns `cb.conjunction()` (no-op) to make the filter optional. Use `root.join("author", JoinType.LEFT)` for predicates on associations.

---

## Auditing

Add `@EnableJpaAuditing`, put `@CreatedDate`/`@LastModifiedDate`/`@CreatedBy`/`@LastModifiedBy` on a `@MappedSuperclass` `BaseEntity` annotated with `@EntityListeners(AuditingEntityListener.class)`. For `@CreatedBy`/`@LastModifiedBy`, implement an `AuditorAware<String>` bean (typically reading `SecurityContextHolder`).

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

See `references/association-mappings.md` for the full pattern. Spring Data-specific reminder: a `Set<Comment>` field on the parent **requires** stable `equals`/`hashCode` on the child (see `references/entity-mapping-checklist.md`), or in-memory `add/remove` won't behave correctly across persistence state transitions.

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
