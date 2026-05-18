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

## Bytecode Enhancement

Hibernate can instrument entity bytecode at build time to improve two distinct behaviors:

### 1. Dirty Checking Enhancement

By default, Hibernate compares all field values at flush time to detect changes — a deep copy is made when the entity is loaded, then compared field-by-field at flush. For wide entities (50+ fields) or high-frequency flush cycles, this deep copy is CPU-intensive.

With bytecode enhancement, Hibernate instruments setters to track dirty state directly — **no snapshot needed, no comparison at flush**.

```xml
<!-- Maven plugin (build-time enhancement) -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <goals><goal>enhance</goal></goals>
            <configuration>
                <enableDirtyTracking>true</enableDirtyTracking>       <!-- key flag -->
                <enableLazyInitialization>true</enableLazyInitialization>
                <enableAssociationManagement>true</enableAssociationManagement>
            </configuration>
        </execution>
    </executions>
</plugin>
```

After enhancement, each entity class implements `SelfDirtinessTracker` — its setters record which fields changed, so flush only examines dirty fields.

**When it matters most:**
- Entities with 20+ fields
- Applications that flush frequently (many small transactions)
- Bulk-read-then-update workflows where most entities are never modified

### 2. Lazy Attribute Loading (Column-Level Lazy)

By default, all columns are loaded eagerly in a single SELECT. For entities with large binary or text columns (e.g., a `body` TEXT field on a Post), you often don't need the large column just to render a list view.

```java
@Entity
public class Post {
    @Id
    private Long id;
    private String title;     // loaded with every SELECT

    @Basic(fetch = FetchType.LAZY)  // only fetched when accessed
    @Column(columnDefinition = "TEXT")
    private String body;           // 10KB–100KB — skip in list queries

    @Basic(fetch = FetchType.LAZY)
    @Column(columnDefinition = "BYTEA")
    private byte[] attachment;     // potentially MB — never load unless needed
}
```

**Requires bytecode enhancement** — `FetchType.LAZY` on `@Basic` is silently ignored without it:

```xml
<configuration>
    <enableLazyInitialization>true</enableLazyInitialization>  <!-- enables @Basic LAZY -->
</configuration>
```

Generated SQL without `body` access:
```sql
SELECT p.id, p.title FROM post p WHERE p.id = ?
-- body loaded only when p.getBody() is called:
SELECT p.body FROM post p WHERE p.id = ?
```

**Use `@Basic(fetch = LAZY)` for:** PDF attachments, HTML body content, large JSON blobs, profile images — anything that bloats the SELECT for list/summary queries.

---

## Hibernate Statistics

`emf.unwrap(SessionFactory.class).getStatistics()` exposes aggregate counters. Key ones to track: `getQueryExecutionCount()` per request (N+1 smell), `getEntityLoadCount()`, L2 cache hit/miss counts, and `getFlushCount()` (unexpected flushes in read paths).

→ See `references/logging-and-monitoring.md` for surfacing these through Micrometer/Prometheus.
