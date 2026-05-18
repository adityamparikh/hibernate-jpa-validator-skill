# Inheritance Strategies

---

## Strategy Comparison

| Strategy | Table Count | Joins | Nullable FKs | Polymorphic Query | Verdict |
|---|---|---|---|---|---|
| `SINGLE_TABLE` | 1 | None | Many nulls | Simple SELECT | **Usually best** |
| `JOINED` | N | Join per level | None | JOIN per type | Use for normalized data |
| `TABLE_PER_CLASS` | N | UNION ALL | None | UNION ALL (slow) | **Avoid** |
| `@MappedSuperclass` | N/A | N/A | N/A | Not polymorphic | Audit fields only |

---

## SINGLE_TABLE (Default and Usually Best)

All subclasses in one table; `@DiscriminatorColumn` identifies the type, `@DiscriminatorValue` on each leaf. Polymorphic queries are a single `SELECT * FROM payment WHERE ...` — zero joins.

**Trade-off:** Subclass-specific columns must be nullable. Add DB-level `@Check` constraints to enforce per-subclass invariants. **Choose when:** hierarchy is stable, few subclasses, fast polymorphic queries matter.

---

## JOINED (Normalized)

Each class has its own table with a FK to the root table (`@PrimaryKeyJoinColumn` is implicit on id). Polymorphic queries generate one `LEFT JOIN` per subclass table:

```sql
SELECT p.*, cc.*, bt.*
FROM payment p
LEFT JOIN credit_card_payment cc ON p.id = cc.id
LEFT JOIN bank_transfer_payment bt ON p.id = bt.id
WHERE ...
```

**Choose when:** subclasses have many non-null specific columns and DB normalization matters. **Avoid for deep hierarchies** (4+ levels) — JOIN count compounds.

---

## TABLE_PER_CLASS — Avoid

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment { ... }
```

Polymorphic query generates:
```sql
SELECT * FROM credit_card_payment
UNION ALL
SELECT * FROM bank_transfer_payment
UNION ALL
SELECT * FROM crypto_payment;
```

`UNION ALL` is hard to optimize, doesn't use indexes well, and gets slower with each new subclass.

**Never use for:** Large tables, polymorphic queries, or when subclasses will grow.

---

## @MappedSuperclass — Not Inheritance

Not JPA inheritance — the superclass is not an entity, just a holder for shared mapped fields (id, audit timestamps, `@Version`) that get inlined into each subclass's table. **You cannot query by the superclass type** — there's no shared table.

---

## Composition Over Inheritance

Prefer composition for behavioral variation:

```java
// ❌ OOP inheritance: PayPalPayment, StripePayment, BankPayment...
// ✅ Composition: Payment + PaymentMethod strategy

@Entity
public class Payment {
    @Id
    @GeneratedValue(...)
    private Long id;

    @Enumerated(EnumType.STRING)
    private PaymentProvider provider;  // STRIPE, PAYPAL, BANK

    @Column(columnDefinition = "jsonb")  // store provider-specific data as JSON
    @Type(JsonType.class)               // Hypersistence Utils JsonType
    private Map<String, Object> providerData;
}
```

**When inheritance makes sense:** True IS-A relationship with shared persistence queries and shared behavior. Not "happens to have similar fields."

---

## @DiscriminatorColumn Best Practices

```java
// ✅ Explicit string discriminator (safe for refactoring)
@DiscriminatorColumn(
    name = "entity_type",
    discriminatorType = DiscriminatorType.STRING,
    length = 31  // 31 chars enough for most class names
)

// ✅ On leaf: explicit value, not class name (class can be renamed)
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment { ... }

// ❌ Don't rely on default discriminator value (= class simple name)
// — breaks when class is renamed or moved
```
