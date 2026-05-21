# Schema Migrations and DDL Validation

The schema is the contract between your application and the database. Treat it as code: version-controlled, tested, deployed deliberately.

---

## Rule: Never Use `ddl-auto=update` or `create-drop` in Production

```properties
# ❌ DANGEROUS — silently alters schema based on entity changes
spring.jpa.hibernate.ddl-auto=update

# ❌ FATAL — drops everything on shutdown
spring.jpa.hibernate.ddl-auto=create-drop

# ✅ CORRECT — startup fails if schema doesn't match entities
spring.jpa.hibernate.ddl-auto=validate

# ✅ ALSO ACCEPTABLE — pure no-op, trust the migration tool entirely
spring.jpa.hibernate.ddl-auto=none
```

`ddl-auto=update` problems:
- Cannot drop columns or constraints (only adds)
- Cannot rename anything
- Order of changes is not guaranteed
- No version history
- Different across replicas if started in different states

---

## Flyway

Add `org.flywaydb:flyway-core` (plus `flyway-database-postgresql` for PostgreSQL 11+ on Flyway 9+).

Migrations live in `src/main/resources/db/migration/` and follow `V<version>__<description>.sql` (two underscores). Version segments are compared **numerically per dot-separated component** — so `V1.10` correctly sorts *after* `V1.2`, and `V1.0 == V1`. Don't assume lexical ordering.

### Properties

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true     # for adopting Flyway on existing DBs
spring.flyway.validate-on-migrate=true     # checksum check — default
spring.flyway.out-of-order=false           # forbid out-of-order migrations
```

### Repeatable Migrations

For views, functions, stored procedures — anything you want to redefine each time it changes:

```
db/migration/
├── V1__schema.sql
├── V2__seed_lookups.sql
└── R__post_summary_view.sql      # repeatable — re-runs when checksum changes
```

`R__` migrations run AFTER all versioned ones, every time their content changes.

---

## Migration Patterns

### Adding a Non-Null Column to a Populated Table

```sql
-- ❌ WRONG — fails if rows exist (no default for non-null)
ALTER TABLE post ADD COLUMN status VARCHAR(20) NOT NULL;

-- ✅ CORRECT — three-step migration over multiple deployments

-- V5__add_post_status_nullable.sql
ALTER TABLE post ADD COLUMN status VARCHAR(20);
UPDATE post SET status = 'ACTIVE' WHERE status IS NULL;

-- (Deploy app reading/writing the new column)

-- V6__make_post_status_not_null.sql
ALTER TABLE post ALTER COLUMN status SET NOT NULL;
```

### Renaming a Column Without Downtime

Five steps over two deployments:
1. Add new column
2. Backfill from old column
3. Application writes both, reads new (deploy)
4. Drop old column (next deploy)
5. Remove dual-write code

### Adding an Index Concurrently (PostgreSQL)

```sql
-- ❌ Locks the table for writes during the build
CREATE INDEX idx_post_status ON post(status);

-- ✅ Non-blocking — must run outside transaction
CREATE INDEX CONCURRENTLY idx_post_status ON post(status);
```

Flyway-specific: `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. Plain SQL comments do **not** influence Flyway — you must opt out of transactional execution explicitly with the `flyway:executeInTransaction` script-config annotation (Flyway 9.19+):

```sql
-- V7__add_post_status_index.sql
-- flyway:executeInTransaction=false
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_post_status ON post(status);
```

Alternatives that achieve the same result:
- Sidecar config file `V7__add_post_status_index.sql.conf` containing `executeInTransaction=false`.
- Global `spring.flyway.execute-in-transaction=false` (only if you really want this for *all* migrations — usually you don't).

---

## DDL Validation Strategy

```properties
# Production
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
```

Boot sequence:
1. Flyway runs migrations (or verifies all are applied)
2. Hibernate `validate` checks each `@Entity` against the schema
3. If any column/type/constraint mismatch → application fails to start

This is the **fail-fast** posture you want — catches drift between code and migrations at deploy time, not at runtime.

### What `validate` Checks

- Each table named in `@Entity` exists
- Each `@Column` exists with correct name
- Column types are compatible (Hibernate uses dialect-specific type mapping)
- Sequences exist if using `SEQUENCE` generators

### What `validate` Does NOT Check

- Indexes (must verify with explicit tests)
- Constraint names (only existence)
- Default values
- Triggers, grants, partitioning

For these, write a dedicated migration test that asserts the actual DDL via `pg_catalog` queries.

---

## Generating an Initial Schema

You should write your migrations by hand, but for the **initial** schema you can let Hibernate generate a starting point:

```properties
spring.jpa.properties.jakarta.persistence.schema-generation.scripts.action=create
spring.jpa.properties.jakarta.persistence.schema-generation.scripts.create-target=schema.sql
```

Run, copy the output to `V1__initial_schema.sql`, hand-edit (add indexes, tweak naming, add comments), then switch to `ddl-auto=validate`.

**Never ship** Hibernate-generated DDL as-is to production — it's missing indexes, has generic constraint names, and won't reflect your DBA's preferences.

---

## Anti-Pattern: Mixing `import.sql` and Migrations

```
# ❌ BAD — these contradict each other
spring.flyway.enabled=true
spring.jpa.properties.hibernate.hbm2ddl.import_files=import.sql
```

`import.sql` is meant for `create` mode. With Flyway, all seed data should be in versioned migrations (or `R__seed.sql` if it must be reapplied).

---

## Multi-Tenant Schemas

Flyway supports per-schema migrations:

```java
@Bean
public FlywayMigrationStrategy migrationStrategy(List<Tenant> tenants) {
    return flyway -> tenants.forEach(t -> {
        Flyway tenantFlyway = Flyway.configure()
            .dataSource(t.dataSource())
            .schemas(t.schemaName())
            .locations("classpath:db/migration")
            .load();
        tenantFlyway.migrate();
    });
}
```

For Hibernate-managed multi-tenancy, see `references/hibernate-features.md` (`@TenantId`).

---

## Testing Migrations

```java
@Test
void migrationFromV3ToV4_preservesData() {
    String url = testContainer.getJdbcUrl();
    String user = testContainer.getUsername();
    String pass = testContainer.getPassword();

    // Step 1: migrate up to V3
    Flyway.configure()
        .dataSource(url, user, pass)
        .target(MigrationVersion.fromVersion("3"))
        .load()
        .migrate();

    seedRowsRepresentingV3Data(jdbc);

    // Step 2: migrate to V4 (a fresh Flyway instance — `configure()` is static)
    Flyway.configure()
        .dataSource(url, user, pass)
        .target(MigrationVersion.fromVersion("4"))
        .load()
        .migrate();

    assertRowsTransformedCorrectly(jdbc);
}
```

Run this in CI for every migration that changes data. Catches lossy or non-idempotent migrations before they hit production.
