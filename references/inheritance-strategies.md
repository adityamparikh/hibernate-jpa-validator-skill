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

All subclasses in one table. Discriminator column identifies the type.

```java
@Entity
@Table(name = "payment")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING, length = 20)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
    @SequenceGenerator(name = "payment_seq", sequenceName = "payment_id_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(nullable = false)
    private Instant createdAt;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    @Column(name = "card_last4", length = 4)
    private String cardLast4;

    @Column(name = "card_brand", length = 20)
    private String cardBrand;
}

@Entity
@DiscriminatorValue("BANK_TRANSFER")
public class BankTransferPayment extends Payment {
    @Column(name = "account_number", length = 30)
    private String accountNumber;
    // card_last4 and card_brand are NULL for this type
}
```

**Polymorphic query — zero joins:**
```sql
SELECT * FROM payment WHERE created_at > '2024-01-01';
```

**Trade-off:** Subclass-specific columns must be nullable. Add `@Check` constraints at DB level to enforce invariants.

**Choose SINGLE_TABLE when:** Hierarchy is stable, subclasses are few, you need fast polymorphic queries.

---

## JOINED (Normalized)

Each class has its own table with a FK to the root table.

```java
@Entity
@Table(name = "payment")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
    @SequenceGenerator(name = "payment_seq", sequenceName = "payment_id_seq", allocationSize = 50)
    private Long id;
    private BigDecimal amount;
}

@Entity
@Table(name = "credit_card_payment")
public class CreditCardPayment extends Payment {
    // Implicitly has: @PrimaryKeyJoinColumn on id
    private String cardLast4;
    private String cardBrand;
}
```

Generated SQL for polymorphic query:
```sql
SELECT p.id, p.amount, cc.card_last4, bt.account_number
FROM payment p
LEFT JOIN credit_card_payment cc ON p.id = cc.id
LEFT JOIN bank_transfer_payment bt ON p.id = bt.id
WHERE p.created_at > '2024-01-01';
```

**Choose JOINED when:** Subclass tables have many non-null-specific columns, DB normalization is required, hierarchy is complex.

**Performance note:** Each polymorphic query adds a JOIN per subclass. For deep hierarchies (4+ levels), this is expensive.

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

Not JPA inheritance — the superclass is not an entity. Use for shared mapped fields.

```java
@MappedSuperclass
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @Version
    private int version;
}

@Entity
public class Post extends BaseEntity { ... }  // gets id, createdAt, updatedAt, version
```

**Cannot** query by `BaseEntity` type — there's no shared table.

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
