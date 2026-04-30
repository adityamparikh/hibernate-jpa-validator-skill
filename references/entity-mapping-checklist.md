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

```java
// ❌ WRONG — IDENTITY disables JDBC batching
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// ❌ WRONG — AUTO is unpredictable per database
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;

// ❌ WRONG — TABLE uses pessimistic locking
@Id
@GeneratedValue(strategy = GenerationType.TABLE)
private Long id;

// ✅ CORRECT — SEQUENCE with pooled optimizer
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "post_seq")
@SequenceGenerator(name = "post_seq", sequenceName = "post_id_seq",
    allocationSize = 50)  // fetch 50 IDs at once — minimal DB roundtrips
private Long id;
```

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

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @Version
    private int version;  // optimistic locking — prevents lost updates
}
```

Flag entities that: modify shared data without `@Version`, lack audit timestamps.

---

## 7. Enum Mapping

```java
// ❌ WRONG — ordinal breaks when enum order changes
@Enumerated(EnumType.ORDINAL)
private Status status;

// ✅ CORRECT — string is resilient to reordering
@Enumerated(EnumType.STRING)
@Column(length = 30)
private Status status;
```

---

## 8. Temporal Types

```java
// ❌ WRONG — java.util.Date is mutable and imprecise
@Temporal(TemporalType.TIMESTAMP)
private Date createdAt;

// ✅ CORRECT — java.time types map natively in Hibernate 6
private Instant createdAt;      // UTC timestamp
private LocalDate birthDate;    // date without time
private LocalDateTime scheduledAt;  // local datetime (no TZ)
private OffsetDateTime eventAt; // with TZ offset
```

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

### Numeric Column Types

```java
// ❌ float/double — floating-point rounding errors
private double price;

// ✅ BigDecimal for money/accounting
@Column(precision = 19, scale = 4)  // 19 digits total, 4 decimal places
private BigDecimal price;

// ✅ Integer types — choose smallest that fits
private int viewCount;       // INT (up to ~2.1 billion)
private long totalViews;     // BIGINT (up to ~9.2 * 10^18)
```

### Binary / UUID Types

```java
// ✅ Native UUID type (PostgreSQL)
@Column(columnDefinition = "uuid")
private UUID externalId;

// ✅ Binary(16) for UUID on MySQL (more efficient than VARCHAR(36))
@Column(columnDefinition = "BINARY(16)")
private UUID externalId;
```

---

## 10. Lombok Compatibility

| Annotation | Safe on Entity? | Notes |
|---|---|---|
| `@Getter` / `@Setter` | ✅ OK | Fine |
| `@ToString` | ⚠️ Risky | Exclude lazy associations — triggers load |
| `@EqualsAndHashCode` | ❌ Avoid | Use manual equals/hashCode |
| `@Data` | ❌ Never | Combines EqualsAndHashCode on all fields |
| `@Builder` | ⚠️ Needs no-arg | Add `@NoArgsConstructor` (JPA requires it) |

```java
// ✅ Safe Lombok on entity
@Entity
@Getter
@Setter
@ToString(exclude = {"comments", "tags"})  // never toString lazy collections
@NoArgsConstructor
public class Post { ... }
```
