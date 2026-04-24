# Hibernate 5 → 6 Migration Guide

Hibernate 6 shipped with Spring Boot 3.0 (aligned with Spring Framework 6). Key breaking changes below.

---

## 1. javax → jakarta Namespace

The biggest mechanical change. Every import must be updated:

```java
// ❌ Hibernate 5 / Spring Boot 2
import javax.persistence.Entity;
import javax.persistence.Column;
import javax.persistence.GeneratedValue;
import javax.persistence.ManyToOne;
import javax.persistence.FetchType;

// ✅ Hibernate 6 / Spring Boot 3
import jakarta.persistence.Entity;
import jakarta.persistence.Column;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.FetchType;
```

Automate with IntelliJ: **Refactor → Migrate Packages and Classes → Java EE to Jakarta EE**

Or with OpenRewrite:
```xml
<dependency>
    <groupId>org.openrewrite.recipe</groupId>
    <artifactId>rewrite-migrate-java</artifactId>
</dependency>
```
Recipe: `org.openrewrite.java.migrate.jakarta.JavaxPersistenceToJakartaPersistence`

---

## 2. Type System Overhaul

### BasicType → UserType Changes

```java
// ❌ Hibernate 5 — UserType interface changed completely
public class JsonType implements UserType {
    @Override
    public Object nullSafeGet(...) { ... }  // different signature
}

// ✅ Hibernate 6 — new UserType<T> generic interface
public class JsonType implements UserType<Map<String, Object>> {
    @Override
    public Map<String, Object> nullSafeGet(ResultSet rs, int pos, WrapperOptions options) { ... }
}
```

**Easiest fix:** Use Hypersistence Utils (Vlad's library) which provides ready-made types for Hibernate 6:
```xml
<dependency>
    <groupId>io.hypersistence</groupId>
    <artifactId>hypersistence-utils-hibernate-63</artifactId>
    <version>3.9.0</version>
</dependency>
```

Provides: `JsonType`, `JsonBinaryType`, `ListArrayType`, `EnumArrayType`, and more.

### @Type Attribute Change

```java
// ❌ Hibernate 5
@Type(type = "json")
private Map<String, Object> metadata;

// ✅ Hibernate 6
@Type(JsonType.class)
private Map<String, Object> metadata;
```

---

## 3. Implicit Naming Strategy Changes

Hibernate 6 changed some default naming conventions.

```properties
# Explicit naming strategy (add to avoid surprises)
spring.jpa.hibernate.naming.physical-strategy=\
    org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy
spring.jpa.hibernate.naming.implicit-strategy=\
    org.springframework.boot.orm.jpa.hibernate.SpringImplicitNamingStrategy
```

**Common surprise:** Join table names and FK column names may differ between Hibernate 5 and 6 with default naming. Always use explicit `@JoinTable(name=...)` and `@JoinColumn(name=...)`.

---

## 4. HQL / JPQL Stricter Parsing

Hibernate 6 uses a new ANTLR-based parser. Queries that worked in H5 may fail:

```java
// ❌ Hibernate 5 allowed — implicitly joined
@Query("SELECT p FROM Post p WHERE p.author.name = :name")
// ✅ Hibernate 6 — explicit JOIN required for cross-entity comparisons
@Query("SELECT p FROM Post p JOIN p.author a WHERE a.name = :name")

// ❌ H5 allowed ambiguous alias
@Query("SELECT p FROM Post p, Author a WHERE p.author = a AND a.name = :name")
// ✅ H6 — use JOIN explicitly
@Query("SELECT p FROM Post p JOIN p.author a WHERE a.name = :name")

// ❌ H5 COUNT(entity) sometimes worked
@Query("SELECT COUNT(p) FROM Post p")  // OK
@Query("SELECT count(*) FROM Post p")  // ❌ H6 stricter — use COUNT(p)
```

---

## 5. Criteria API Changes

```java
// ❌ Hibernate 5 — CriteriaQuery type could be raw
CriteriaQuery cq = cb.createQuery();

// ✅ Hibernate 6 — must be typed
CriteriaQuery<Post> cq = cb.createQuery(Post.class);

// ❌ H5 — JoinType on root fetch was inconsistent
root.fetch("author");

// ✅ H6 — explicit LEFT or INNER required for predictable behavior
root.fetch("author", JoinType.LEFT);
```

---

## 6. @Where → @SQLRestriction

```java
// ❌ Deprecated in Hibernate 6.3
@Where(clause = "deleted = false")
public class Post { ... }

// ✅ Replacement
@SQLRestriction("deleted = false")
public class Post { ... }

// For collections (also deprecated):
@WhereJoinTable(clause = "active = true")  // ❌
@SQLJoinTableRestriction("active = true")   // ✅
```

---

## 7. GenerationType.AUTO Behavior

Hibernate 5 with PostgreSQL: `AUTO` → usually `SEQUENCE`
Hibernate 6 with PostgreSQL: `AUTO` → `SEQUENCE` (but uses `hibernate_sequence` by default, shared across ALL entities)

```java
// ❌ Don't rely on AUTO — explicit is always better
@GeneratedValue(strategy = GenerationType.AUTO)

// ✅ Always explicit
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "post_seq")
@SequenceGenerator(name = "post_seq", sequenceName = "post_id_seq", allocationSize = 50)
```

---

## 8. Removed / Renamed APIs

| Hibernate 5 | Hibernate 6 |
|---|---|
| `Session.load()` | `Session.getReference()` |
| `Session.get()` | `Session.find()` (JPA standard) |
| `Criteria` (legacy) | Removed — use JPA Criteria or HQL |
| `@TypeDef` | Removed — use `@Type(MyType.class)` inline |
| `@AnyMetaDef` | `@Any` with inline meta |
| `AbstractType` | Removed — implement `UserType<T>` |
| `CompositeUserType` (old) | New `CompositeUserType<J>` generic |
| `@Cache(region=...)` | Same, but `region` semantics adjusted |

---

## 9. Spring Boot 3 Alignment

If migrating from Spring Boot 2.x to 3.x:

```xml
<!-- Spring Boot 3 BOM pulls Hibernate 6.x automatically -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.x.y</version>
</parent>
```

Key Spring Data JPA changes in SB3:
- `CrudRepository.findById()` still returns `Optional` — no change
- `JpaRepository.getById()` deprecated → use `getReferenceById()`
- `JpaRepository.getOne()` removed → use `getReferenceById()`

```java
// ❌ SB2
Post post = postRepository.getById(id);      // deprecated
Post post = postRepository.getOne(id);       // removed

// ✅ SB3
Post post = postRepository.getReferenceById(id);  // returns proxy, no SQL until accessed
Post post = postRepository.findById(id).orElseThrow();  // loads eagerly
```

---

## Migration Checklist

- [ ] Replace `javax.persistence.*` with `jakarta.persistence.*`
- [ ] Update `@Type` to use class reference instead of string name
- [ ] Replace `@Where` with `@SQLRestriction`
- [ ] Replace `Session.load()` with `Session.getReference()`, `Session.get()` with `Session.find()`
- [ ] Replace `getById()`/`getOne()` with `getReferenceById()` or `findById()`
- [ ] Verify all JPQL queries with new strict parser (run test suite)
- [ ] Explicit `@GeneratedValue` strategy — never `AUTO`
- [ ] Update custom `UserType` implementations to `UserType<T>` interface
- [ ] Add Hypersistence Utils if using JSON/array types
- [ ] Verify naming strategy matches your existing schema DDL
