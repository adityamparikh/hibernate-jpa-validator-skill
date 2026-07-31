# Migrating Hibernate 6 → 7

**Last verified: 2026-07** against the official Hibernate ORM 7.0 migration guide.
Version-sensitive: re-check https://docs.hibernate.org/orm/7.0/migration-guide/migration-guide.html
before asserting any specific removal.

**Baselines:** Java **17+**, **Jakarta Persistence 3.2**. Hibernate 7 ships with
Spring Boot 4.x. If you are still on Boot 3.x you are on Hibernate 6 — do the Boot
upgrade first, not the ORM upgrade in isolation.

Read this in two passes. The **silent** changes are the dangerous half: they compile
cleanly and change runtime behaviour. The **loud** changes are removals — annoying,
but the compiler finds them for you.

---

## Silent behavioural changes — review these first

### 1. Native query temporal types changed from `java.sql` to `java.time`

A native query that returned `java.sql.Timestamp` now returns `java.time.Instant`
(and equivalently for date/time). Any cast, `instanceof`, or mapper on the result
breaks at runtime, not compile time.

```properties
# Restore the old behaviour while you migrate the call sites
hibernate.query.native.prefer_jdbc_datetime_types=true
```

Prefer fixing the call sites — `java.time` is the better target. Use the flag as a
temporary bridge, not a destination.

### 2. `StatelessSession` now uses the second-level cache by default

Previously it bypassed L2 entirely. If you used `StatelessSession` for bulk work
*specifically to avoid* polluting the cache, that assumption is now inverted.

```java
statelessSession.setCacheMode(CacheMode.IGNORE);  // restore the old bypass
```

### 3. `StatelessSession` ignores `hibernate.jdbc.batch_size`

The global batch-size setting no longer applies to it. Batching silently degrades to
one statement per row — the classic "why did my bulk job get 50× slower" symptom.

```java
statelessSession.setJdbcBatchSize(50);  // now required, explicitly
```

### 4. Cascade `PERSIST` is no longer auto-enabled on `@Id` / `@MapsId` associations

Hibernate used to silently enable `cascade=PERSIST` for association fields annotated
`@Id` or `@MapsId`. That undocumented behaviour is gone. Code relying on it now fails
to persist the association — declare the cascade explicitly.

### 5. Detached-entity operations now throw

- A detached instance reached by a cascaded persist raises
  `jakarta.persistence.EntityExistsException` unless you `merge` it first.
- Refreshing a detached entity throws `IllegalArgumentException` (JPA-spec conformance).
- Acquiring a lock on a detached entity is likewise disallowed.

### 6. DDL / column-type changes — these alter generated schema

| Change | Impact |
|---|---|
| `char` / `Character` now map to `varchar(1)`, not `char(1)` | Schema validation fails against an existing `char(1)` column |
| Oracle `float`/`double` → `binary_float`/`binary_double` | Precision and storage change |
| Oracle timestamp precision → 9 (nanos); SQL Server → 7 | Validation mismatch on existing columns |
| Arrays on DB2, SAP HANA, SQL Server, Sybase ASE → `SqlTypes.XML_ARRAY` | Storage format change |
| Arrays on MySQL / MariaDB → `SqlTypes.JSON_ARRAY` | Storage format change |

**With `ddl-auto=validate` (which you should be running — see SKILL.md), these surface
as startup failures rather than data corruption.** That is the good outcome. Reconcile
via a Flyway migration rather than by loosening validation.

### 7. Immutable-entity update/delete queries now throw

```properties
# Only if you genuinely need the old permissiveness
hibernate.query.immutable_entity_update_query_handling_mode=allow
```

---

## Loud changes — the compiler finds these

### Removed `Session` methods

| Removed | Use instead |
|---|---|
| `Session#load` | `Session#getReference` |
| `Session#save` | `Session#persist` |
| `Session#update` | `Session#merge` |
| `Session#saveOrUpdate` | `#persist` or `#merge`, chosen explicitly |
| `Session#delete` | `Session#remove` |
| `Session#get` | `Session#find` |
| `Session#refresh(String, Object)` and the `LockOptions` overload | the JPA `refresh` overloads |
| `Session.LockRequest` | `LockOptions` |

### Removed `CascadeType` values

`CascadeType.SAVE_UPDATE` → `PERSIST` and/or `MERGE`.
`CascadeType.DELETE` → `REMOVE`.

### Removed annotations

`@Persister`, `@Proxy`, `@SelectBeforeUpdate`, `@Loader`, Hibernate's `@Table`,
`@ForeignKey`, `@Index`, `@IndexColumn`, `@GeneratorType`, `@LazyToOne`,
`@LazyCollection`, and Hibernate's `@Where`, `@WhereJoinTable` and `@OrderBy`.

The filtering ones have direct successors that this skill already documents in
`hibernate-features.md`:

| Removed | Replacement |
|---|---|
| `@Where` | `@SQLRestriction` |
| `@WhereJoinTable` | `@SQLJoinTableRestriction` |
| Hibernate `@OrderBy` | `@SQLOrder` (or JPA's own `@jakarta.persistence.OrderBy`) |
| `@Target` | `@TargetEmbeddable` |

`@SQLRestriction` was introduced in 6.3 as the preferred form — in 7 it is the only form.

### Renamed / changed

- `CacheModeType` → `CacheMode`
- `@Cache#include` → `@Cache#includeLazy`
- `@DynamicInsert#value` / `@DynamicUpdate#value` removed — presence of the annotation
  now means `true`
- `org.hibernate.Metamodel` → `org.hibernate.metamodel.model.domain.JpaMetamodel`
- `Query#setOrder` removed (was incubating)
- `EmptyInterceptor` removed; `Interceptor`'s deprecated methods (`onLoad`,
  `onFlushDirty`, `onSave`, `onDelete`, the `onCollection*` family, `findDirty`,
  `getEntity`) removed
- `UserType` / `CompositeUserType` signatures changed to drop layer-breaking SPI types
- The `org.hibernate.graph` package was re-engineered with breaking type signatures

### Queries must declare a result type

A query with an implicit `SELECT` and no explicit result type is no longer accepted.
Give it a result class or an explicit select clause.

```java
// ❌ rejected in 7
session.createQuery("from Book").getResultList();

// ✅
session.createQuery("from Book", Book.class).getResultList();
```

### Removed settings and pools

- Settings: `hibernate.mapping.precedence`, `hibernate.allow_refresh_detached_entity`
- Connection pools: **Vibur, Proxool and UCP support dropped.** HikariCP is unaffected
  and remains the Spring Boot default — see `connection-pooling.md`.

### SPI (only if you write extensions)

`Configurable#configure` takes `GeneratorCreationContext` instead of `ServiceRegistry`;
`Integrator#integrate(Metadata, SessionFactoryImplementor, SessionFactoryServiceRegistry)`,
`MetamodelImplementor`, `AdditionalJaxbMappingProducer` and `MetadataContributor` are
removed (use `MappingMetamodel`/`JpaMetamodel` and `AdditionalMappingContributor`).

### Maven plugin

Coordinates changed to `org.hibernate.orm:hibernate-maven-plugin`. Note that
`enableLazyInitialization` and `enableDirtyTracking` defaulted to `false` in 7.0 — a
regression, fixed in 7.1. If you rely on build-time enhancement, set them explicitly
or move to 7.1+.

---

## Suggested order of work

1. **Get to Java 17+ and Spring Boot 4** first. Hibernate 7 is what Boot 4 manages;
   upgrading the ORM alone against Boot 3 is a fight you don't need.
2. **Compile.** Fix the loud removals — mostly mechanical `Session` method renames and
   `@Where` → `@SQLRestriction`.
3. **Run with `ddl-auto=validate`** against a real database via Testcontainers. The DDL
   changes in §6 surface here, loudly, before they reach an environment that matters.
4. **Audit `StatelessSession` usages** for the cache and batch-size changes (§2, §3) —
   these are silent and performance-shaped, so no test will fail, it will just get slow.
5. **Grep native queries** for `java.sql.*` handling (§1).
6. **Only then** consider the compatibility flags, and treat each as a dated TODO
   rather than a setting.
