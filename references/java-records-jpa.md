# Java Records with JPA

Records are `final`, have only `final` components, and have no synthesizable no-arg constructor. All three violate JPA's basic contract. The non-obvious part is *where* records still work, and where Hibernate 6 has quietly added support.

---

## 1. Records cannot be `@Entity`

Three blockers, any one of which is fatal:

```java
// ❌ Will not work as an entity
@Entity
public record Post(@Id Long id, String title) {}
```

1. `final` class → Hibernate can't generate a `LAZY` proxy subclass. `ManyToOne(fetch = LAZY)` to a record-entity is impossible.
2. No no-arg constructor → Hibernate's `Instantiator` fails. (Hibernate 6 *can* use the canonical constructor for embeddables, but not entities.)
3. Final components → there's no way for Hibernate to update fields on flush / merge / refresh.

Records also cannot be `@MappedSuperclass` for the same reasons.

---

## 2. Records as `@Embeddable` work in Hibernate 6.2+

This is the one place Hibernate quietly added record support. The canonical constructor is used during hydration; the record stays immutable. Updating an embedded record means replacing the whole instance on the parent entity — which dirty-checking handles fine.

```java
public record Money(BigDecimal amount, String currency) {}

@Entity
public class Order {
    @Id Long id;
    @Embedded Money total;   // ✅ works in Hibernate 6.2+
}

// To "update":
order.setTotal(new Money(newAmount, "USD"));   // replace, not mutate
```

Hibernate 5 / older 6.x: records as `@Embeddable` fail. Stick to a regular class with private setters.

---

## 3. Records as `@IdClass` are excellent

Records are immutable with auto-generated `equals`/`hashCode` based on all components — exactly what JPA wants for a composite key class. This is one of the cleanest record use-cases:

```java
public record OrderLineId(Long orderId, Long lineNumber) implements Serializable {}

@Entity
@IdClass(OrderLineId.class)
public class OrderLine {
    @Id Long orderId;
    @Id Long lineNumber;
    // ...
}
```

`@EmbeddedId` with a record works in Hibernate 6.2+; older versions need a regular class.

---

## 4. Records are perfect for Spring Data **class** projections — not interface projections

Spring Data's two projection styles work differently:

```java
// ✅ Class projection: Spring Data invokes the canonical constructor
public record PostSummary(Long id, String title, Instant createdAt) {}

interface PostRepository extends JpaRepository<Post, Long> {
    List<PostSummary> findAllProjectedBy();
}
```

```java
// ❌ Interface projection: Spring Data generates a proxy implementing the interface.
//    Records are final — they cannot be proxied.
//    What you actually want for "interface projections" is a regular interface.
public interface PostSummary {
    Long getId();
    String getTitle();
}
```

If you mix these up, you'll get cryptic `Could not introspect record class` errors at startup.

---

## 5. JPQL constructor expressions require the fully-qualified class name

This isn't record-specific, but records make it sting more because the pattern looks so natural.

```java
// ✅ Must be FQN, not just `PostSummary`
@Query("SELECT new com.example.dto.PostSummary(p.id, p.title, p.createdAt) FROM Post p")
List<PostSummary> findSummaries();
```

Hibernate 6.1+ supports `select` returning records via the canonical constructor automatically when the result class is a record — but only with native SQL + `@SqlResultSetMapping` or with `EntityManager.createQuery(jpql, PostSummary.class)` style. The JPQL `new` form is still required for portable code.

---

## 6. NULL columns + primitive components = NPE during hydration

A record's canonical constructor is invoked with the projected column values. If a column is nullable and the record component is a primitive (`long`, `int`), Hibernate's unboxing crashes with `NullPointerException` deep inside hydration — far from the actual problem.

```java
// ❌ If `view_count` can be NULL, every NULL row crashes hydration
public record PostStats(Long id, long viewCount) {}

// ✅ Use the boxed type to allow null
public record PostStats(Long id, Long viewCount) {}
```

---

## 7. Native query result mapping with records

`@SqlResultSetMapping` with `@ConstructorResult` targets records natively — handy when the query is hand-tuned SQL but you want a typed result without a custom row mapper.

```java
@SqlResultSetMapping(
    name = "PostSummaryMapping",
    classes = @ConstructorResult(
        targetClass = PostSummary.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "title", type = String.class)
        }
    )
)
```

---

## 8. `with…()` methods don't help persistence

Records don't have setters; some teams generate `withTitle(...)` style methods or use libraries to do so. These return a **new instance**. Calling `post.withTitle("Hi")` on a managed entity returns a detached object Hibernate doesn't know about. The dirty-checking-friendly path simply doesn't exist for record-shaped entities — another reason records aren't entities.

---

## 9. Summary table

| Use case | Works? | Notes |
|---|---|---|
| `@Entity` | ❌ Never | Final, immutable, no no-arg ctor |
| `@MappedSuperclass` | ❌ Never | Same blockers as entity |
| `@Embeddable` | ✅ Hibernate 6.2+ | Replace instance to "update"; older Hibernate fails |
| `@IdClass` | ✅ Always | Immutable + auto equals/hashCode is exactly right |
| `@EmbeddedId` | ✅ Hibernate 6.2+ | Older versions need a regular class |
| Spring Data class projection | ✅ Always | Canonical constructor invoked by Spring Data |
| Spring Data interface projection | ❌ Use interface | Records can't be proxied |
| JPQL `new ...` constructor result | ✅ Always | FQN required |
| Native SQL `@ConstructorResult` | ✅ Always | Map columns to components |
| `@AttributeConverter` input/output | ✅ Always | Records are great for JSON / value-object converters |
