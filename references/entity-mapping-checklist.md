# Entity Mapping Checklist

Reference for Section B of SKILL.md. Run every item against every entity under review.

---

## 1. Class-Level Annotations

| Check | Good | Bad |
|---|---|---|
| Table name explicitly set | `@Table(name = "post")` | Relying on implicit naming (breaks across naming strategies) |
| Schema specified if multi-schema | `@Table(schema = "blog")` | Missing — creates cross-schema confusion |
| Reference data marked immutable | `@Immutable` present | Missing on country/currency/lookup tables |
| Frequently-partial-updated entities | `@DynamicUpdate` present | Missing — Hibernate updates all columns |
| Cache annotation for shared reads | `@Cache(usage = READ_ONLY)` | Missing on `@Immutable` entities |

---

## 2. Identifier Rules

Use `SEQUENCE` with a pooled optimizer (`allocationSize=50`). Never `IDENTITY` (disables JDBC batching), `AUTO` (unpredictable per database), or `TABLE` (pessimistic locking).

→ See `references/identifier-strategies.md` for the full code and why each alternative fails.

---

## 3. equals/hashCode

This is **the #1 correctness issue** in Hibernate applications.

### Why Generated ID Alone Fails

```java
// Before persist: id = null
// After persist:  id = 1
// Two transient Post objects with id=null are equal — breaks Sets!

Post p1 = new Post("First");   // id = null
Post p2 = new Post("Second");  // id = null
Set<Post> set = new HashSet<>();
set.add(p1);
set.add(p2);  // bucket collision — both hash to same bucket
// set.size() == 1 or 2, undefined!
```

### Pattern 1: Business Key (Preferred when natural key exists)

```java
@Entity
public class Post {
    @NaturalId
    @Column(unique = true, nullable = false, updatable = false)
    private String slug;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Post)) return false;
        Post post = (Post) o;
        return Objects.equals(slug, post.slug);
    }

    @Override
    public int hashCode() {
        return Objects.hash(slug);
    }
}
```

### Pattern 2: Stable hashCode with Generated ID (When No Business Key)

```java
@Override
public boolean equals(Object o) {
    if (o == null || getClass() != o.getClass()) return false;
    Post post = (Post) o;
    return id != null && id.equals(post.id);  // never equal when transient
}

@Override
public int hashCode() {
    return getClass().hashCode();  // same across all states — no collision on id changes
}
```

### Anti-Patterns to Flag

| Pattern | Problem |
|---|---|
| IDE-generated with all fields | Changes when fields change — breaks persistence |
| Lombok `@EqualsAndHashCode` (default) | Uses all fields — same problem |
| `@EqualsAndHashCode(onlyExplicitlyIncluded=true)` without `@IncludeField` | Equivalent to identity equals — may be OK |
| `@Data` on entity | Includes all fields in equals/hashCode — almost always wrong |

---

## 4. Column Definitions Checklist

```java
@Entity
public class Post {

    @Id  // ...sequence generator...
    private Long id;

    @Column(
        name = "title",
        nullable = false,       // ✅ matches DB NOT NULL constraint
        length = 500            // ✅ explicit length, not default 255
    )
    private String title;

    @Column(
        name = "slug",
        nullable = false,
        unique = true,          // ✅ DB unique constraint
        updatable = false,      // ✅ slug should never change
        length = 600
    )
    private String slug;

    @Column(
        name = "body",
        columnDefinition = "TEXT"   // ✅ explicit type for large text
    )
    private String body;

    @Column(
        name = "created_at",
        nullable = false,
        updatable = false       // ✅ audit field never changes
    )
    private Instant createdAt;

    @Column(name = "price", precision = 19, scale = 4)  // ✅ for BigDecimal
    private BigDecimal price;
}
```

**Flag these column definition gaps:**
- `String` field with no `length` (default 255 may truncate)
- Nullable FK column without `nullable = false` matching DB
- `BigDecimal` without `precision` and `scale`
- `createdAt`/`insertedAt` without `updatable = false`
- Natural key columns without `updatable = false`

---

## 5. Index Hints

```java
@Entity
@Table(
    name = "post",
    indexes = {
        @Index(name = "idx_post_author_id", columnList = "author_id"),   // FK column
        @Index(name = "idx_post_slug", columnList = "slug", unique = true),
        @Index(name = "idx_post_status_created", columnList = "status, created_at")
    }
)
public class Post { ... }
```

**Flag missing indexes on:**
- Every `@ManyToOne` FK column (author_id, category_id, etc.)
- Columns used in WHERE clauses in common queries
- Columns used in ORDER BY with large tables

---

## 6. Audit Fields Pattern

Use a `@MappedSuperclass` `BaseEntity` with `@CreatedDate`/`@LastModifiedDate` (Spring Data auditing) + `@Version` (optimistic locking). Flag entities that modify shared data without `@Version` or lack audit timestamps.

---

## 7. Enum Mapping

Always `@Enumerated(EnumType.STRING)` — `ORDINAL` breaks the instant anyone reorders or inserts an enum value.

---

## 8. Temporal Types

Use `java.time` types directly (`Instant`, `LocalDate`, `LocalDateTime`, `OffsetDateTime`) — Hibernate 6 maps them natively. `@Temporal` + `java.util.Date` is a legacy carryover; remove it.

---

## 9. Column Type Optimization

Choosing the right SQL type has performance and correctness implications.

### Avoid CLOB/BLOB When VARCHAR/VARBINARY Works

```java
// ❌ WRONG — CLOB is an out-of-row LOB by default on many DBs
@Lob
private String description;
// On PostgreSQL: Hibernate may use oid type → separate pg_largeobject table
// On MySQL: TEXT/MEDIUMTEXT stored out-of-row for large values

// ✅ CORRECT — use TEXT or VARCHAR for reasonably sized strings
@Column(columnDefinition = "TEXT")    // stored inline in PostgreSQL
private String description;

@Column(length = 4000)               // VARCHAR(4000) — inline, with length enforcement
private String summary;
```

**Rule:** Only use `@Lob` / `CLOB` when content genuinely exceeds 4–8 KB. For most descriptions, comments, and body text, `TEXT` or `VARCHAR(n)` is faster (no LOB detouring) and simpler.

### @Nationalized — Unicode Columns (SQL Server / Oracle)

On SQL Server and Oracle, `VARCHAR` is non-Unicode. For international content:

```java
// ❌ On SQL Server: VARCHAR can't store CJK/Arabic/emoji
@Column(length = 200)
private String name;

// ✅ NVARCHAR on SQL Server, NCLOB on Oracle
@Nationalized
@Column(length = 200)
private String name;
```

On PostgreSQL and MySQL (utf8mb4), `@Nationalized` is a no-op — all VARCHAR columns are Unicode. Only needed for SQL Server and Oracle.

### Numeric and Binary Types

- Money/accounting: `BigDecimal` with explicit `precision`/`scale`, never `float`/`double`.
- UUID: native `uuid` columnDefinition on PostgreSQL; `BINARY(16)` on MySQL (vs. wasteful `VARCHAR(36)`).

---

## 10. Lombok / Records / Kotlin Data Classes

Quick summary — see `references/lombok-records-kotlin.md` for the full set of non-obvious gotchas.

### Lombok on `@Entity`

| Annotation | Safe on Entity? | Notes |
|---|---|---|
| `@Getter` / `@Setter` | ✅ OK | Don't expose `@Id` setter publicly |
| `@ToString` | ⚠️ Always `exclude` associations | Silent N+1 in logs, LazyInit outside tx |
| `@EqualsAndHashCode` | ❌ Avoid | Even with `id` only — transient entities collide (id=null) |
| `@Data` | ❌ Never | Bundles the two dangerous annotations above |
| `@Value` | ❌ Never | Final + immutable — no Hibernate proxy, no hydration |
| `@Builder` | ⚠️ Needs `@NoArgsConstructor` | And `@Builder.Default` on initialized collection fields |
| `@SuperBuilder` | ⚠️ OK | Use for `@MappedSuperclass` hierarchies |
| `@FieldNameConstants` | ✅ Recommended | Type-safe `Sort`/Criteria/Specification |

```java
// ✅ Safe Lombok on entity
@Entity
@Getter
@Setter
@ToString(exclude = {"comments", "tags"})  // never toString lazy collections
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
public class Post {
    @OneToMany(mappedBy = "post")
    @Builder.Default                          // ← without this, builder() sets to null
    private List<Comment> comments = new ArrayList<>();
}
```

### Java Records

Records are **final**, **immutable**, and lack a no-arg constructor — incompatible with `@Entity` / `@MappedSuperclass`. They shine elsewhere:

| Use case | Verdict |
|---|---|
| `@Entity` / `@MappedSuperclass` | ❌ Never |
| `@Embeddable` / `@EmbeddedId` | ✅ Hibernate 6.2+ only |
| `@IdClass` composite key | ✅ Best-in-class — immutable, auto equals/hashCode |
| Spring Data **class** projection | ✅ Canonical constructor invoked by Spring Data |
| Spring Data **interface** projection | ❌ Use an `interface` — records can't be proxied |
| JPQL `new com.example.PostDTO(...)` | ✅ FQN required |
| Native `@SqlResultSetMapping` `@ConstructorResult` | ✅ Works |
| Primitive component for nullable column | ❌ NPE on hydration — use boxed type |

### Kotlin Data Classes

Same problems as Lombok `@Data` (auto equals/hashCode/toString on all properties) plus records' problems (final by default, no no-arg constructor).

| Pattern | Verdict |
|---|---|
| `data class` as `@Entity` | ❌ Never — `copy()` creates accidental detached entities |
| Regular `class` as `@Entity` with `kotlin-jpa` plugin | ✅ Required combination |
| `kotlin-noarg` alone (no `kotlin-allopen`) | ❌ Class still final → LAZY proxies impossible |
| `Long` (non-null) on `@Id` | ❌ Use `Long?` — `0L` breaks transient check |
| Non-null Kotlin type, nullable DB column | ❌ Bypasses Kotlin's null check via reflection |
| `data class` as `@Embeddable` (with `kotlin-jpa`) | ⚠️ OK — "update" by replacing the whole instance |
| `data class` as DTO projection | ✅ Same as Java record |
| `@JvmInline value class` field | ❌ Erased at JVM level — needs `AttributeConverter` |
| `by lazy` for collection | ❌ Not Hibernate lazy — never reflects DB state |

```kotlin
// ✅ Idiomatic Kotlin JPA entity
@Entity
class Post(
    var title: String,
) {
    @Id @GeneratedValue
    var id: Long? = null
        private set

    @OneToMany(mappedBy = "post")
    val comments: MutableList<Comment> = mutableListOf()

    override fun equals(other: Any?): Boolean {
        if (other == null || Hibernate.getClass(this) != Hibernate.getClass(other)) return false
        other as Post
        return id != null && id == other.id
    }

    override fun hashCode(): Int = Hibernate.getClass(this).hashCode()
}
```

→ See `references/lombok-records-kotlin.md` for full code examples and the underlying reasons.
