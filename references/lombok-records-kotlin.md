# Lombok, Java Records, and Kotlin Data Classes with JPA

Non-obvious gotchas when mixing modern Java/Kotlin abstractions with Hibernate. These three families of features all promise concise code, and all three collide with JPA's three core requirements:

1. **Mutability** — Hibernate hydrates fields after construction (reflection or generated setters).
2. **No-arg constructor** — JPA spec requires one (Hibernate uses `newInstance` then sets fields).
3. **Non-final classes** — Hibernate subclasses entities at runtime for `LAZY` proxies and dirty-check bytecode enhancement.

Lombok, records, and Kotlin data classes each violate at least one of those by default. The fixes below are mostly about knowing which guard rails to put back.

---

## Part 1 — Lombok Gotchas

### L1. `@Data` on an entity is almost always wrong

`@Data` bundles `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, and `@RequiredArgsConstructor`. Two of those (`@EqualsAndHashCode`, `@ToString`) default to including **all fields**, which on an entity means lazy associations are forced.

```java
// ❌ Looks innocent. Three different ways to break.
@Entity
@Data
public class Post {
    @Id @GeneratedValue Long id;
    String title;
    @OneToMany(mappedBy = "post") List<Comment> comments = new ArrayList<>();
}

// 1. equals/hashCode includes `comments` → forces lazy load every comparison.
// 2. Set<Post> with two transient (id=null) instances collide in the same bucket.
// 3. toString() iterates `comments` → silent N+1 in log statements.
```

### L2. `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + id is still wrong

Most teams discover `@Data` is bad, then "fix" it by including only the id. This is the same bug — two transient entities both have `id = null` and compare equal.

```java
// ❌ Two new Posts (id=null) are equal. Sets break.
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Post {
    @EqualsAndHashCode.Include
    @Id @GeneratedValue Long id;
}
```

Use a `@NaturalId` business key, or write equals/hashCode by hand using the "stable hashCode" pattern (`hashCode = getClass().hashCode()`, equals returns false when `id == null`). See `references/entity-mapping-checklist.md` §3.

### L3. `@ToString` silently triggers lazy loading

This is the most common production log explosion. A `log.info("Saving {}", post)` call deep inside a transaction looks harmless. The generated `toString()` iterates every field including `@OneToMany` collections — N additional SELECTs per log line. Outside the transaction you get `LazyInitializationException` instead.

```java
// ✅ Always exclude collections and entity associations from toString
@ToString(exclude = {"comments", "tags", "author"})
// Or use field-level exclusion:
@ToString
public class Post {
    @ToString.Exclude @OneToMany(mappedBy = "post") List<Comment> comments;
    @ToString.Exclude @ManyToOne Author author;
}
```

### L4. `@Builder` deletes the no-arg constructor

Lombok's `@Builder` synthesizes an all-args constructor and **removes the implicit no-arg constructor**. Hibernate then fails at startup or first load with `InstantiationException`. The fix is explicit `@NoArgsConstructor`, almost always at `PROTECTED` so application code can't bypass the builder:

```java
@Entity
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA needs this
@AllArgsConstructor(access = AccessLevel.PRIVATE)    // builder uses this
public class Post { ... }
```

### L5. `@Builder.Default` is required for initialized collections

This one bites silently. A field with an initializer like `private List<Comment> comments = new ArrayList<>()` will be set to **null** by the generated builder unless you add `@Builder.Default`. The first `post.getComments().add(c)` then NPEs, or worse, Hibernate inserts a null collection and you discover it later.

```java
// ❌ builder() produces post.comments == null
@Builder
public class Post {
    @OneToMany(mappedBy = "post")
    private List<Comment> comments = new ArrayList<>();
}

// ✅ builder() respects the initializer
@Builder
public class Post {
    @OneToMany(mappedBy = "post")
    @Builder.Default
    private List<Comment> comments = new ArrayList<>();
}
```

### L6. `@SuperBuilder` for `@MappedSuperclass`

Plain `@Builder` doesn't traverse a superclass. If you have `BaseEntity` with audit fields and use `@Builder` on the subclass, audit fields are unreachable through the builder. `@SuperBuilder` walks the hierarchy:

```java
@MappedSuperclass
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public abstract class BaseEntity {
    @CreatedDate Instant createdAt;
    @Version Long version;
}

@Entity
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Post extends BaseEntity { ... }
```

### L7. `@Value` is final — never use on an entity

`@Value` is `@Data` for immutables: makes the class `final`, all fields `final`, no setters. Hibernate cannot subclass it for proxies, cannot mutate fields after construction. Use `@Value` for DTO projections only — or just use a `record`.

### L8. `@FieldNameConstants` is a hidden win for type-safe queries

Non-obvious in the *positive* direction. Lombok's `@FieldNameConstants` generates a `Fields` inner class with `public static final String` constants — perfect for Spring Data `Sort`, Criteria queries, and `@Query` parameter names that survive refactors:

```java
@Entity
@FieldNameConstants
public class Post {
    private String title;
    private Instant createdAt;
}

// In a repository or service:
Sort.by(Post.Fields.createdAt).descending();
criteriaBuilder.equal(root.get(Post.Fields.title), "Hello");
```

### L9. Lombok-generated getters bypass Hibernate enhancement when using field access

If your entity uses field access (`@Id` on a field, the default in most apps), Hibernate reads fields via reflection. Lombok-generated getters still work for application code, but **Hibernate never calls them** — meaning any side-effect logic you put inside a getter (e.g. lazy init guard) is ignored. Don't put logic in getters on entities, regardless of whether they're hand-written or Lombok.

### L10. Lombok summary table

| Annotation | Safe on Entity? | Why / Fix |
|---|---|---|
| `@Getter` | ✅ OK | No effect on Hibernate (uses field access) |
| `@Setter` | ⚠️ OK with care | Don't expose `@Id` setter publicly |
| `@ToString` | ⚠️ Risky | Always exclude associations + collections |
| `@EqualsAndHashCode` | ❌ Avoid | Even with `id` only: transient entities collide |
| `@Data` | ❌ Never | Bundles the two dangerous annotations above |
| `@Value` | ❌ Never | Final + immutable: no proxies, no hydration |
| `@Builder` | ⚠️ OK | Add `@NoArgsConstructor` + `@Builder.Default` |
| `@SuperBuilder` | ⚠️ OK | Use for `@MappedSuperclass` hierarchies |
| `@AllArgsConstructor` | ⚠️ OK | Make it `PRIVATE` so callers go through builder |
| `@NoArgsConstructor` | ✅ Required | Make it `PROTECTED` to discourage misuse |
| `@FieldNameConstants` | ✅ Recommended | Type-safe Sort/Criteria/Specification |
| `@Slf4j` | ✅ OK | Logger is `static final`, no Hibernate impact |
| `@With` | ❌ Avoid on entity | Creates copies — detaches from persistence context |

---

## Part 2 — Java Records Gotchas

Records are `final`, have only `final` components, and have no synthesizable no-arg constructor. All three violate JPA's basic contract. The non-obvious part is *where* records still work, and where Hibernate 6 has quietly added support.

### R1. Records cannot be `@Entity`

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

### R2. Records as `@Embeddable` work in Hibernate 6.2+

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

### R3. Records as `@IdClass` are excellent

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

### R4. Records are perfect for Spring Data **class** projections — but not interface projections

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

### R5. JPQL constructor expressions require the fully-qualified class name

This isn't record-specific, but records make it sting more because the pattern looks so natural.

```java
// ✅ Must be FQN, not just `PostSummary`
@Query("SELECT new com.example.dto.PostSummary(p.id, p.title, p.createdAt) FROM Post p")
List<PostSummary> findSummaries();
```

Hibernate 6.1+ supports `select` returning records via the canonical constructor automatically when the result class is a record — but only with native SQL + `@SqlResultSetMapping` or with `EntityManager.createQuery(jpql, PostSummary.class)` style. The JPQL `new` form is still required for portable code.

### R6. NULL columns + primitive components in records = NPE during hydration

A record's canonical constructor is invoked with the projected column values. If a column is nullable and the record component is a primitive (`long`, `int`), Hibernate's unboxing crashes with `NullPointerException` deep inside hydration — far from the actual problem.

```java
// ❌ If `view_count` can be NULL, every NULL row crashes hydration
public record PostStats(Long id, long viewCount) {}

// ✅ Use the boxed type to allow null
public record PostStats(Long id, Long viewCount) {}
```

### R7. Native query result mapping with records

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

### R8. `@With` / record `with…()` methods don't apply to persistence

Records don't have setters; some teams generate `withTitle(...)` style methods or use libraries to do so. These return a **new instance**. Calling `post.withTitle("Hi")` on a managed entity returns a detached object Hibernate doesn't know about. The dirty-checking-friendly path simply doesn't exist for record-shaped entities — another reason records aren't entities.

### R9. Records summary table

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

---

## Part 3 — Kotlin Data Class Gotchas

Kotlin data classes share Lombok's `@Data` problem (auto equals/hashCode/toString on all properties) **and** Java records' problem (final class, no no-arg constructor by default). The Kotlin compiler plugins exist to paper over the JPA contract, but they only get you to a working app — not necessarily a correct one.

### K1. Use `kotlin-jpa`, not just `kotlin-noarg`

`kotlin-noarg` synthesizes a no-arg constructor for annotated classes. `kotlin-allopen` removes `final` from annotated classes so Hibernate can subclass them. `kotlin-jpa` is the meta-plugin that configures both for the standard JPA annotations (`@Entity`, `@MappedSuperclass`, `@Embeddable`):

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.jpa") version "1.9.0"  // ← noarg + allopen tuned for JPA
    kotlin("plugin.spring") version "1.9.0"
}
```

Without `kotlin-jpa`: `InstantiationException` on startup (no no-arg ctor) and `LazyInitializationException` on every `@ManyToOne(fetch = LAZY)` (final class, no proxy).

### K2. **Don't** use `data class` for entities

This is the headline rule and gets ignored constantly because data classes feel idiomatic. The auto-generated `equals`/`hashCode`/`toString` use **every property**, including lazy associations. Same problems as Lombok `@Data`:

```kotlin
// ❌ data class as entity
@Entity
data class Post(
    @Id @GeneratedValue val id: Long? = null,
    val title: String,
    @OneToMany(mappedBy = "post") val comments: MutableList<Comment> = mutableListOf()
)
// equals() forces `comments` to load every comparison.
// toString() N+1s in log statements.
// copy() returns a new instance with the same id — detached and dangerous.
```

```kotlin
// ✅ regular class with manual equals/hashCode
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

The `Hibernate.getClass(...)` calls unwrap proxies so a `Post` and its `Post$HibernateProxy` compare equal — a regular `javaClass` check fails for lazy proxies.

### K3. `data class .copy()` creates an accidental detached entity

The most insidious bug. `copy()` returns a new instance with the same `@Id`. If you mutate the copy and try to persist it, you get a duplicate or a `NonUniqueObjectException`. If you call `repository.save(post.copy(title = "new"))`, Spring Data does a MERGE that wins over the managed entity — silently undoing other concurrent updates.

### K4. Non-null property + nullable column = surprise NPE

Kotlin's type system promises `String` is never null, but Hibernate populates fields via reflection — bypassing the null check. The NPE only appears when application code dereferences the field, far from where the bad data lives.

```kotlin
// ❌ Column is NULLABLE in the DB but Kotlin promises non-null
@Column(nullable = true)
var email: String = ""

// ✅ Match nullability across DB and Kotlin
@Column(nullable = true)
var email: String? = null

// ✅ Or enforce NOT NULL at the DB and Kotlin level
@Column(nullable = false)
var email: String = ""
```

Use `ddl-auto=validate` (or migration-driven schema) to keep these in sync.

### K5. `Long?` for `@Id`, not `Long`

A non-null `Long` defaults to `0L`, meaning a "new" entity has id `0L` instead of `null`. Hibernate's "is this transient?" check (`id == null`) misfires, leading to MERGE-instead-of-INSERT behavior on save. Always nullable:

```kotlin
@Id @GeneratedValue
var id: Long? = null
```

### K6. `val` properties: Hibernate can still set them, but you can't

Hibernate uses reflection to set fields directly, so `val` "works" — it's hydrated on load. But your application code can't update a `val`, so you can't actually use Hibernate's dirty-checking on it. Mark mutable fields `var`. Use `val` only for fields that are truly immutable after construction (e.g. `@Id`, audit-created timestamps with `updatable = false`).

### K7. `lateinit var` is useful for non-null fields populated by Hibernate

For non-null fields that aren't set in the primary constructor (e.g. computed in `@PostLoad` or set during a multi-step builder), `lateinit var` avoids the awkward `var foo: String = ""` placeholder:

```kotlin
@Entity
class Post {
    @Id @GeneratedValue
    var id: Long? = null

    @Column(nullable = false)
    lateinit var slug: String  // set by builder before persist
}
```

`lateinit` throws `UninitializedPropertyAccessException` rather than NPE, which is easier to diagnose.

### K8. `@JvmInline value class` doesn't map cleanly

Value classes are erased to their underlying type at the JVM level (`Email(val value: String)` becomes `String` in bytecode). Hibernate sees the underlying type and won't invoke the wrapper. You need an `AttributeConverter`:

```kotlin
@JvmInline
value class Email(val value: String)

@Converter(autoApply = true)
class EmailConverter : AttributeConverter<Email, String> {
    override fun convertToDatabaseColumn(attribute: Email?) = attribute?.value
    override fun convertToEntityAttribute(dbData: String?) = dbData?.let { Email(it) }
}
```

Even with this, generic types (`List<Email>`) and using value classes in queries (`findByEmail(email: Email)`) often fail. Pragmatic advice: avoid value classes in entity fields.

### K9. Property delegation (`by lazy`) is not Hibernate lazy

`by lazy { ... }` is a Kotlin computed-once delegate; Hibernate lazy loading is a proxy-based mechanism. They are unrelated. Don't use `by lazy` for `@OneToMany` collections — it captures the initial empty value and never reflects later DB state.

### K10. `data class` is fine for DTO projections

The problems above are specific to *entities*. Data classes are excellent for Spring Data class projections (same as Java records):

```kotlin
data class PostSummary(val id: Long, val title: String, val createdAt: Instant)

interface PostRepository : JpaRepository<Post, Long> {
    fun findAllProjectedBy(): List<PostSummary>
}
```

Same caveat as records: interface projections need an `interface`, not a `data class`.

### K11. `@Embeddable` data class needs `kotlin-jpa`

Embeddables also need a no-arg constructor and non-finality. `kotlin-jpa` configures both for `@Embeddable`:

```kotlin
@Embeddable
data class Money(
    @Column(nullable = false) val amount: BigDecimal,
    @Column(length = 3, nullable = false) val currency: String,
)
```

To "update" an embedded value: replace the whole object on the parent. Same shape as records.

### K12. Sealed classes for inheritance — apply `kotlin-allopen`

Sealed hierarchies are popular for domain modeling. To map them as JPA inheritance (`SINGLE_TABLE` / `JOINED`), the sealed parent and its subtypes need `kotlin-allopen` (the `kotlin-jpa` plugin handles this when they're annotated `@Entity` / `@MappedSuperclass`).

```kotlin
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
sealed class Payment(...) { ... }

@Entity @DiscriminatorValue("CARD")
class CardPayment(...) : Payment(...)
```

Often it's cleaner to map the data layer with regular classes and convert to a sealed-class domain model in the service layer.

### K13. Kotlin summary table

| Pattern | Verdict | Notes |
|---|---|---|
| `data class` as `@Entity` | ❌ Never | All-fields equals/hashCode/toString + `copy()` foot-guns |
| Regular `class` as `@Entity` | ✅ With `kotlin-jpa` | Manual equals/hashCode with `Hibernate.getClass` |
| `data class` as `@Embeddable` | ⚠️ OK | Needs `kotlin-jpa`; "update" by replacement |
| `data class` as DTO projection | ✅ Always | Canonical constructor used by Spring Data |
| `val` on `@Id` | ✅ OK | Hibernate hydrates via reflection |
| `val` on mutable column | ❌ Avoid | App code can't update; dirty checking unusable |
| `var` on mutable column | ✅ Always | Standard |
| `Long` (non-null) on `@Id` | ❌ Never | Use `Long?` so transient state is `null`, not `0L` |
| `lateinit var` on non-null field | ✅ OK | Better diagnostics than `""` placeholder |
| `@JvmInline value class` field | ❌ Avoid | Erased at JVM level; needs converter, often breaks queries |
| `by lazy` for collection field | ❌ Never | Not Hibernate lazy; never refreshes |
| Sealed class hierarchy as JPA inheritance | ⚠️ OK | Needs `kotlin-allopen`; often better in domain layer |
| `kotlin-noarg` plugin only | ❌ Incomplete | Fixes ctor but classes still final → no lazy proxies |
| `kotlin-jpa` plugin | ✅ Required | The standard combo |

---

## Cross-Cutting Rule

For all three of these abstractions: **the more you let the language/library generate equality, toString, and constructors based on every field, the more JPA contracts you break.** The fix is always the same shape — opt out of "all fields" defaults, write explicit equals/hashCode (or use a `@NaturalId`), exclude associations from `toString`, and add the missing no-arg / non-final guarantees that JPA requires.
