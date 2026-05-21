# Kotlin Data Classes with JPA

Kotlin data classes share Lombok `@Data`'s problem (auto equals/hashCode/toString on all properties) **and** Java records' problem (final class, no no-arg constructor by default). The Kotlin compiler plugins exist to paper over the JPA contract, but they only get you to a working app — not necessarily a correct one.

---

## 1. Use `kotlin-jpa`, not just `kotlin-noarg`

`kotlin-noarg` synthesizes a no-arg constructor for annotated classes. `kotlin-allopen` removes `final` from annotated classes so Hibernate can subclass them. `kotlin-jpa` is the meta-plugin that configures both for the standard JPA annotations (`@Entity`, `@MappedSuperclass`, `@Embeddable`):

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.jpa") version "1.9.0"  // ← noarg + allopen tuned for JPA
    kotlin("plugin.spring") version "1.9.0"
}
```

Without `kotlin-jpa`: `InstantiationException` on startup (no no-arg ctor) and `LazyInitializationException` on every `@ManyToOne(fetch = LAZY)` (final class, no proxy).

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

---

## 6. `val` properties: Hibernate can still set them, but you can't

Hibernate uses reflection to set fields directly, so `val` "works" — it's hydrated on load. But your application code can't update a `val`, so you can't actually use Hibernate's dirty-checking on it. Mark mutable fields `var`. Use `val` only for fields that are truly immutable after construction (e.g. `@Id`, audit-created timestamps with `updatable = false`).

---

## 7. `lateinit var` is useful for non-null fields populated by Hibernate

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

## 8. `@JvmInline value class` doesn't map cleanly

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

## 9. Property delegation (`by lazy`) is not Hibernate lazy

`by lazy { ... }` is a Kotlin computed-once delegate; Hibernate lazy loading is a proxy-based mechanism. They are unrelated. Don't use `by lazy` for `@OneToMany` collections — it captures the initial empty value and never reflects later DB state.

---

## 10. `data class` is fine for DTO projections

The problems above are specific to *entities*. Data classes are excellent for Spring Data class projections (same as Java records):

```kotlin
data class PostSummary(val id: Long, val title: String, val createdAt: Instant)

interface PostRepository : JpaRepository<Post, Long> {
    fun findAllProjectedBy(): List<PostSummary>
}
```

Same caveat as records: interface projections need an `interface`, not a `data class`.

---

## 11. `@Embeddable` data class needs `kotlin-jpa`

Embeddables also need a no-arg constructor and non-finality. `kotlin-jpa` configures both for `@Embeddable`:

```kotlin
@Embeddable
data class Money(
    @Column(nullable = false) val amount: BigDecimal,
    @Column(length = 3, nullable = false) val currency: String,
)
```

To "update" an embedded value: replace the whole object on the parent. Same shape as records.

---

## 12. Sealed classes for inheritance — apply `kotlin-allopen`

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

---

## 13. Summary table

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
