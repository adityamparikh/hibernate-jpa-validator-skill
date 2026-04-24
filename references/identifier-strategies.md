# Identifier Strategies

---

## Strategy Comparison

| Strategy | JDBC Batching | Roundtrips | Recommendation |
|---|---|---|---|
| `SEQUENCE` + pooled-lo | ✅ Yes | Minimal | **Use this** |
| `SEQUENCE` (allocationSize=1) | ✅ Yes | 1 per insert | OK, but wasteful |
| `IDENTITY` | ❌ **Breaks batching** | 1 per insert | Avoid for bulk writes |
| `TABLE` | ❌ Pessimistic lock | Many | Never |
| `AUTO` | Database-dependent | Unknown | Never |

### Why IDENTITY Breaks JDBC Batching

With `IDENTITY`, the database generates the ID **after** the INSERT completes. Hibernate must execute each INSERT individually and read back the generated key before it can associate the entity. This makes `executeBatch()` impossible.

With `SEQUENCE`, Hibernate fetches IDs ahead of time (using `allocationSize`), then batches all INSERTs without any roundtrip per row.

```sql
-- IDENTITY: Hibernate must do this per entity
INSERT INTO post (title) VALUES ('Hello');
SELECT LAST_INSERT_ID();  -- or RETURNING id

-- SEQUENCE (allocationSize=50): one sequence call for 50 inserts
SELECT nextval('post_id_seq');  -- returns 50
-- Hibernate pre-allocates IDs 50..99, then batches:
INSERT INTO post (id, title) VALUES (50, 'A'), (51, 'B'), ...  -- one round-trip
```

---

## SEQUENCE with Pooled Optimizer (Recommended)

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "post_seq")
@SequenceGenerator(
    name = "post_seq",
    sequenceName = "post_id_seq",
    allocationSize = 50    // must match INCREMENT BY in DDL
)
private Long id;
```

**DDL (must match allocationSize):**
```sql
CREATE SEQUENCE post_id_seq INCREMENT BY 50 START WITH 1;
```

**Pooled vs Pooled-Lo:**
- Default (pooled): DB sequence value = next HIGH end. E.g., sequence returns 50 → allocates IDs 1–50.
- Pooled-Lo: DB sequence value = LOW end. E.g., sequence returns 1 → allocates 1–50. More intuitive.

To use pooled-lo:
```java
@SequenceGenerator(
    name = "post_seq",
    sequenceName = "post_id_seq",
    allocationSize = 50,
    parameters = @Parameter(name = SequenceStyleGenerator.OPT_PARAM, value = "pooled-lo")
)
```

**allocationSize tuning:**
- 50 is a good default
- High-volume tables (millions/day): increase to 100–500
- Low-volume tables: 10–20 is fine

---

## UUID Strategies

### Random UUID (avoid for indexed columns)

```java
// ❌ Random — causes B-tree index fragmentation
@Id
@GeneratedValue(strategy = GenerationType.UUID)
@UuidGenerator(style = UuidGenerator.Style.RANDOM)
private UUID id;
```

Random UUIDs scatter inserts across the entire B-tree, causing page splits and cache misses. For a 10M-row table, random UUID inserts can be 10–50x slower than sequential IDs.

### Time-Based UUID (v7 — preferred for UUIDs)

```java
// ✅ Time-ordered — minimal index fragmentation
@Id
@UuidGenerator(style = UuidGenerator.Style.TIME)  // UUID v1 variant
private UUID id;
```

Or use a custom generator for UUID v7 (RFC 9562 — monotonic within millisecond):
```java
@Id
@Column(columnDefinition = "uuid", updatable = false, nullable = false)
private UUID id = Generators.timeBasedEpochGenerator().generate();  // UUID v7 via java-uuid-generator
```

**When to use UUID over Long:**
- Distributed systems without central ID authority
- IDs need to be unpredictable (security)
- Data sharding by entity ID

---

## @NaturalId

Use when an entity has a stable business key separate from the surrogate PK.

```java
@Entity
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "book_seq")
    @SequenceGenerator(name = "book_seq", sequenceName = "book_id_seq", allocationSize = 50)
    private Long id;

    @NaturalId                                    // Hibernate-managed natural key
    @Column(name = "isbn", unique = true, nullable = false, updatable = false)
    private String isbn;
}
```

**Benefits of @NaturalId:**
- Hibernate caches `isbn → id` mapping in second-level cache
- Enables direct lookup via `session.byNaturalId(Book.class).using("isbn", isbn).load()`
- Works with `@NaturalIdCache` for ultra-fast lookups

```java
// Natural ID lookup — hits L2 cache if configured
Book book = entityManager.unwrap(Session.class)
    .byNaturalId(Book.class)
    .using("isbn", "978-0-13-468599-1")
    .load();
```

---

## Composite Keys

### @IdClass (simpler, less invasive)

```java
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals, hashCode required!
}

@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    @Id
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;
}
```

### @EmbeddedId (more OO, preferred for complex PKs)

```java
@Embeddable
public class OrderItemId implements Serializable {
    @Column(name = "order_id")
    private Long orderId;
    @Column(name = "product_id")
    private Long productId;
    // equals, hashCode required!
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("orderId")
    @JoinColumn(name = "order_id")
    private Order order;
}
```

**Rules for composite keys:**
- Implement `Serializable`
- Implement `equals`/`hashCode` based on all ID fields
- Prefer `@EmbeddedId` when the ID is conceptually a first-class value object
- Never use mutable fields as composite key components
