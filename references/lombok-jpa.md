# Lombok with JPA

Non-obvious gotchas when using Lombok on Hibernate entities. Lombok's defaults — generating equality, `toString`, and constructors that include every field — collide with JPA's requirements (no-arg constructor, mutable fields, non-final classes for `LAZY` proxies).

---

## 1. `@Data` on an entity is almost always wrong

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

---

## 2. `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + id is still wrong

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

---

## 3. `@ToString` silently triggers lazy loading

The most common production log explosion. A `log.info("Saving {}", post)` call deep inside a transaction looks harmless. The generated `toString()` iterates every field including `@OneToMany` collections — N additional SELECTs per log line. Outside the transaction you get `LazyInitializationException` instead.

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

---

## 4. `@Builder` deletes the no-arg constructor

Lombok's `@Builder` synthesizes an all-args constructor and **removes the implicit no-arg constructor**. Hibernate then fails at startup or first load with `InstantiationException`. The fix is explicit `@NoArgsConstructor`, almost always at `PROTECTED` so application code can't bypass the builder:

```java
@Entity
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA needs this
@AllArgsConstructor(access = AccessLevel.PRIVATE)    // builder uses this
public class Post { ... }
```

---

## 5. `@Builder.Default` is required for initialized collections

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

---

## 6. `@SuperBuilder` for `@MappedSuperclass`

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

---

## 7. `@Value` is final — never use on an entity

`@Value` is `@Data` for immutables: makes the class `final`, all fields `final`, no setters. Hibernate cannot subclass it for proxies, cannot mutate fields after construction. Use `@Value` for DTO projections only — or just use a `record`.

---

## 8. `@FieldNameConstants` is a hidden win for type-safe queries

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

---

## 9. Lombok-generated getters bypass Hibernate enhancement

If your entity uses field access (`@Id` on a field, the default in most apps), Hibernate reads fields via reflection. Lombok-generated getters still work for application code, but **Hibernate never calls them** — meaning any side-effect logic you put inside a getter (e.g. lazy init guard) is ignored. Don't put logic in getters on entities, regardless of whether they're hand-written or Lombok.

---

## 10. Summary table

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
