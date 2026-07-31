# Kotlin Data Classes with JPA

Kotlin data classes share Lombok `@Data`'s problem (auto equals/hashCode/toString on all properties) **and** Java records' problem (final class, no no-arg constructor by default). The Kotlin compiler plugins exist to paper over the JPA contract, but they only get you to a working app — not necessarily a correct one.

---

## 1. `kotlin-jpa` alone is not enough — you also need `kotlin-allopen`

`kotlin-noarg` synthesizes a no-arg constructor for annotated classes. `kotlin-allopen` removes `final` from annotated classes so Hibernate can subclass them. `kotlin-jpa` is a **no-arg-only** wrapper: it auto-configures `@Entity`, `@MappedSuperclass`, and `@Embeddable` for constructor generation, but it does **not** touch finality. `kotlin-spring` is the equivalent all-open wrapper, but it's scoped to Spring annotations (`@Component`, `@Configuration`, `@Transactional`, ...) — it does not open `@Entity` by default either. All-open for JPA has to be configured explicitly:

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.jpa") version "2.0.20"     // no-arg ctor for @Entity/@MappedSuperclass/@Embeddable
    kotlin("plugin.allopen") version "2.0.20" // makes those same classes non-final
    kotlin("plugin.spring") version "2.0.20"  // separate: opens Spring-managed beans, not entities
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

Without `kotlin-jpa`: `InstantiationException` on startup (no no-arg ctor). With `kotlin-jpa` but without the `allOpen` block above: no ctor error, but every `@ManyToOne(fetch = LAZY)` throws `LazyInitializationException` or silently fetches eagerly, because the entity class is still `final` and Hibernate's default proxy-based lazy loading can't subclass it.

---

## 2. **Don't** use `data class` for entities

This is the headline rule and gets ignored constantly because data classes feel idiomatic. The auto-generated `equals`/`hashCode`/`toString` use **every property**, including lazy associations.

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

---

## 3. `data class .copy()` creates an accidental detached entity

The most insidious bug. `copy()` returns a new instance with the same `@Id`. If you mutate the copy and try to persist it, you get a duplicate or a `NonUniqueObjectException`. If you call `repository.save(post.copy(title = "new"))`, Spring Data does a MERGE that wins over the managed entity — silently undoing other concurrent updates.

---

## 4. Non-null property + nullable column = surprise NPE

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

---

## 5. `Long?` for `@Id`, not `Long`

A non-null `Long` defaults to `0L`, meaning a "new" entity has id `0L` instead of `null`. Hibernate's "is this transient?" check (`id == null`) misfires, leading to MERGE-instead-of-INSERT behavior on save. Always nullable:

```kotlin
@Id @GeneratedValue
var id: Long? = null
```

Why it often "works" anyway, and why that's not a reason to keep it: Hibernate treats a primary-key value of `0` differently per method. `persist()` expects an unassigned id and throws if one looks already set; `merge()` treats `0` as "not yet assigned," generates a new key, and inserts a fresh row. Spring Data's `save()` picks `merge()` whenever it thinks the entity isn't new (its `isNew()` check is `id == null`), so a primitive `Long` id often still produces a correct insert — but at the cost of an extra `SELECT` per save (merge loads-then-merges), and it will silently *update* an existing row if one genuinely has id `0`. Prefer `Long?` and let `persist()` insert directly.

---

## 6. `val` properties: Hibernate can still set them, but you can't

Hibernate uses reflection to set fields directly, so `val` "works" — it's hydrated on load. But your application code can't update a `val`, so you can't actually use Hibernate's dirty-checking on it. Mark mutable fields `var`. Use `val` only for fields that are truly immutable after construction (e.g. `@Id`, audit-created timestamps with `updatable = false`).

---

## 7. Annotations on properties use field access by default

Most Kotlin JPA code puts annotations directly on primary-constructor properties (`@Id val id: Long? = null`). Hibernate treats this as **field access** by default: it reads and writes the backing field via reflection and never calls your getter/setter, even if you wrote one. That's the common case and it's fine — the gotcha shows up only if you deliberately opt into property access:

```kotlin
@Entity
@Access(AccessType.PROPERTY)   // Hibernate now calls accessors instead of touching fields
class Post {
    @Id @GeneratedValue
    var id: Long? = null       // must be `var` — a `val` has no generated setter
}
```

Under `AccessType.PROPERTY`, every annotated property must be `var`. Kotlin never emits a setter for `val`, so Hibernate has no method to call — decompile the class and `setId` simply isn't there.

**Kotlin 2.2 also changed what an annotation on a property targets.** Before 2.2, `@Column val name: String` in a primary constructor applied the annotation to the constructor **parameter only** — not the backing field or the generated getter — which could cause JPA or bean-validation annotations to be silently ignored depending on framework internals. Kotlin 2.2 added a compiler option (IntelliJ IDEA offers to enable it) that applies such annotations to the parameter *and* the property/field by default. On an older Kotlin version, if an annotation seems to have no effect, check whether an explicit `@field:`/`@get:` use-site target is needed.

---

## 8. `lateinit var` is useful for non-null fields populated by Hibernate

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

---

## 9. `@JvmInline value class` doesn't map cleanly

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

---

## 10. Property delegation (`by lazy`) is not Hibernate lazy

`by lazy { ... }` is a Kotlin computed-once delegate; Hibernate lazy loading is a proxy-based mechanism. They are unrelated. Don't use `by lazy` for `@OneToMany` collections — it captures the initial empty value and never reflects later DB state.

---

## 11. `data class` is fine for DTO projections

The problems above are specific to *entities*. Data classes are excellent for Spring Data class projections (same as Java records):

```kotlin
data class PostSummary(val id: Long, val title: String, val createdAt: Instant)

interface PostRepository : JpaRepository<Post, Long> {
    fun findAllProjectedBy(): List<PostSummary>
}
```

Same caveat as records: interface projections need an `interface`, not a `data class`.

---

## 12. `@Embeddable` data class needs `kotlin-jpa`

Embeddables need a no-arg constructor, which `kotlin-jpa` configures. Unlike entities, embeddables aren't proxied by Hibernate (they're loaded inline with the owning entity's row), so they don't need `kotlin-allopen`/non-finality:

```kotlin
@Embeddable
data class Money(
    @Column(nullable = false) val amount: BigDecimal,
    @Column(length = 3, nullable = false) val currency: String,
)
```

To "update" an embedded value: replace the whole object on the parent. Same shape as records.

---

## 13. Sealed classes for inheritance — apply `kotlin-allopen`

Sealed hierarchies are popular for domain modeling. To map them as JPA inheritance (`SINGLE_TABLE` / `JOINED`), the sealed parent and its subtypes need `kotlin-allopen` configured for `@Entity`/`@MappedSuperclass` — `kotlin-jpa` alone (no-arg only) does not open them; see §1.

```kotlin
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
sealed class Payment(...) { ... }

@Entity @DiscriminatorValue("CARD")
class CardPayment(...) : Payment(...)
```

Often it's cleaner to map the data layer with regular classes and convert to a sealed-class domain model in the service layer.

---

## 14. Summary table

| Pattern | Verdict | Notes |
|---|---|---|
| `data class` as `@Entity` | ❌ Never | All-fields equals/hashCode/toString + `copy()` foot-guns |
| Regular `class` as `@Entity` | ✅ With `kotlin-jpa` + `kotlin-allopen` | Manual equals/hashCode with `Hibernate.getClass` |
| `data class` as `@Embeddable` | ⚠️ OK | Needs `kotlin-jpa` only (no proxying, so no `allopen`); "update" by replacement |
| `data class` as DTO projection | ✅ Always | Canonical constructor used by Spring Data |
| `val` on `@Id` | ✅ OK | Hibernate hydrates via reflection |
| `val` on mutable column | ❌ Avoid | App code can't update; dirty checking unusable |
| `var` on mutable column | ✅ Always | Standard |
| `Long` (non-null) on `@Id` | ❌ Never | Use `Long?`; `merge()` papers over `0` with an extra SELECT, `persist()` throws |
| Property-annotated field, no `@Access` | ✅ Default | Field access — getters/setters bypassed, which is usually fine |
| `@Access(AccessType.PROPERTY)` on a `val` | ❌ Never | No generated setter — Hibernate can't write the property |
| `lateinit var` on non-null field | ✅ OK | Better diagnostics than `""` placeholder |
| `@JvmInline value class` field | ❌ Avoid | Erased at JVM level; needs converter, often breaks queries |
| `by lazy` for collection field | ❌ Never | Not Hibernate lazy; never refreshes |
| Sealed class hierarchy as JPA inheritance | ⚠️ OK | Needs `kotlin-allopen` configured for `@Entity`; often better in domain layer |
| `kotlin-noarg` plugin only | ❌ Incomplete | Fixes ctor but classes still final → no lazy proxies |
| `kotlin-jpa` plugin only | ❌ Incomplete | No-arg only — does not open classes, same lazy-proxy problem as above |
| `kotlin-jpa` + `kotlin-allopen` (configured for JPA annotations) | ✅ Required | The correct combo — no-arg ctor and non-final classes |
