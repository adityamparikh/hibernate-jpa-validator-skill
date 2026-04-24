# Hibernate-Specific Features

Hibernate 6.x / Spring Boot 3.x features beyond standard JPA.

---

## @DynamicUpdate / @DynamicInsert

```java
// Without @DynamicUpdate — always generates full UPDATE
// UPDATE account SET name=?, email=?, balance=?, plan=?, updated_at=? WHERE id=?

@Entity
@DynamicUpdate  // only dirty columns
public class Account {
    private String name;
    private String email;
    private BigDecimal balance;
    private String plan;
    private Instant updatedAt;
}

// Now: UPDATE account SET balance=? WHERE id=?
```

**When to apply:**
- 10+ columns in entity
- Most updates modify 1–3 fields
- High-traffic entities where reducing I/O matters

**@DynamicInsert** — similar but for INSERT: only includes non-null fields. Useful when many columns have DB defaults you want to preserve.

```java
@Entity
@DynamicInsert  // omit null fields from INSERT, let DB defaults fire
public class Post {
    private String status;  // DB default = 'DRAFT' — not overridden by INSERT NULL
}
```

---

## @Immutable

```java
@Entity
@Immutable  // Hibernate skips dirty checking on flush
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
@Table(name = "country")
public class Country {
    @Id
    private String code;
    private String name;
    private String continent;
}
```

**Effect:**
- No dirty checking on flush (saves CPU for every flush cycle)
- Attempts to update/delete throw exception
- Pairs with `READ_ONLY` cache — safest, fastest cache strategy

**Apply to:** country, currency, timezone, any lookup table loaded once at startup.

---

## @Subselect — Map a View as an Entity

```java
@Entity
@Subselect("""
    SELECT
        a.id,
        a.name,
        COUNT(p.id) AS post_count,
        MAX(p.created_at) AS last_post_date
    FROM author a
    LEFT JOIN post p ON p.author_id = a.id
    GROUP BY a.id, a.name
    """)
@Synchronize({"author", "post"})  // flush these tables before querying this view
@Immutable
public class AuthorStats {
    @Id
    private Long id;
    private String name;
    private long postCount;
    private Instant lastPostDate;
}
```

`@Synchronize` tells Hibernate: before running a query against `AuthorStats`, flush pending changes to `author` and `post` tables. Prevents stale reads.

---

## @Filter / @FilterDef — Dynamic WHERE Clauses

Apply reusable filters without changing every query:

```java
// Define filter on entity
@Entity
@FilterDef(
    name = "activeFilter",
    parameters = @ParamDef(name = "status", type = String.class)
)
@Filter(name = "activeFilter", condition = "status = :status")
public class Post { ... }

// Activate in service
Session session = entityManager.unwrap(Session.class);
session.enableFilter("activeFilter").setParameter("status", "ACTIVE");

// Now all Post queries include: AND status = 'ACTIVE'
List<Post> posts = postRepository.findAll();  // only ACTIVE posts returned

// Disable when done
session.disableFilter("activeFilter");
```

**Use case:** Multi-tenant filtering, soft-delete visibility, feature flags per session.

---

## @SQLRestriction (Hibernate 6.3+)

Static WHERE clause baked into every query against an entity — replaces deprecated `@Where`:

```java
@Entity
@SQLRestriction("deleted = false")  // always appended to queries
public class Post {
    private boolean deleted = false;
}
// SELECT * FROM post WHERE ... AND deleted = false
```

For collections:
```java
@OneToMany(mappedBy = "post")
@SQLRestriction("status <> 'SPAM'")  // filters comments globally
private List<Comment> comments;
```

---

## @SoftDelete (Hibernate 6.4+)

First-class soft delete support — Hibernate handles the DELETE translation:

```java
@Entity
@SoftDelete  // uses 'deleted' boolean column by default
public class Post { ... }

// OR with custom column:
@Entity
@SoftDelete(columnName = "is_deleted")
public class Post { ... }

// OR with timestamp:
@Entity
@SoftDelete(strategy = SoftDeleteType.DELETED, columnName = "deleted_at")
public class Post { ... }
```

Now `entityManager.remove(post)` generates:
```sql
UPDATE post SET deleted = true WHERE id = ?
```
And all queries automatically append `AND deleted = false`.

---

## @TenantId (Hibernate 6.0+)

Column-based multi-tenancy:

```java
@Entity
public class Post {
    @TenantId
    @Column(name = "tenant_id", nullable = false, updatable = false)
    private String tenantId;
}

// Spring Boot config:
@Bean
CurrentTenantIdentifierResolver<String> tenantResolver() {
    return new CurrentTenantIdentifierResolver<>() {
        @Override
        public String resolveCurrentTenantIdentifier() {
            return TenantContext.getCurrentTenant();  // from ThreadLocal
        }
        @Override
        public boolean validateExistingCurrentSessions() { return false; }
    };
}
```

Hibernate automatically:
- Populates `tenant_id` on INSERT
- Appends `AND tenant_id = ?` to all queries
- Prevents cross-tenant data leakage

---

## Custom Types and @JdbcTypeCode

```java
// Map a JSON column to a Java Map/Object (requires Hypersistence Utils or Hibernate 6 native)
@Column(columnDefinition = "jsonb")
@JdbcTypeCode(SqlTypes.JSON)
private Map<String, Object> metadata;

// Map PostgreSQL array
@Column(columnDefinition = "text[]")
@JdbcTypeCode(SqlTypes.ARRAY)
private String[] tags;

// Enum stored as PostgreSQL native enum
@Column(columnDefinition = "post_status")
@JdbcTypeCode(SqlTypes.NAMED_ENUM)
private PostStatus status;
```

---

## Hibernate Statistics

```java
@Autowired
private EntityManagerFactory emf;

public void logStats() {
    Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
    log.info("Entity load count: {}", stats.getEntityLoadCount());
    log.info("Query execution count: {}", stats.getQueryExecutionCount());
    log.info("L2 hit ratio: {}/{}", stats.getSecondLevelCacheHitCount(),
        stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount());
    log.info("Connection count: {}", stats.getConnectCount());
}
```

Key metrics to track:
- `getQueryExecutionCount()` per request → detect N+1
- `getEntityLoadCount()` → how many entities loaded
- `getSecondLevelCacheHitCount()` vs `MissCount()` → cache effectiveness
- `getFlushCount()` → unexpected flushes in read paths
