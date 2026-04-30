# hibernate-jpa-validator

A Claude Code skill for comprehensive Hibernate/JPA analysis, inspired by [Vlad Mihalcea's Hypersistence Optimizer](https://vladmihalcea.com/hypersistence-optimizer/).

This skill brings Hypersistence Optimizer's philosophy to AI-assisted code review: when you ask Claude to review an entity, it runs the **full checklist** — not just what you asked about.

---

## What It Covers

| Area | Details |
|---|---|
| **Entity Mapping** | equals/hashCode correctness, @NaturalId, access strategies, @Immutable, @DynamicUpdate, column definitions, indexes, Lombok compatibility |
| **Identifier Strategies** | SEQUENCE > IDENTITY > TABLE > AUTO, pooled/pooled-lo optimizers, UUID strategies (random vs time-ordered), composite keys |
| **Association Mappings** | @ManyToOne (LAZY always), @OneToMany with mappedBy, @ManyToMany with Set (bag vs set semantics), @OneToOne with @MapsId, bidirectional sync methods, cascade rules |
| **N+1 Detection** | How N+1 occurs, JOIN FETCH, @EntityGraph, @BatchSize, @Fetch(SUBSELECT), MultipleBagFetchException, Hibernate statistics, datasource-proxy |
| **Query Optimization** | JPQL best practices, Criteria API, DTO projections, keyset pagination, query plan cache, Blaze Persistence, jOOQ, @QueryHints |
| **Batch Processing** | jdbc.batch_size, order_inserts/updates, flush/clear pattern, StatelessSession, JPQL bulk UPDATE/DELETE, Spring chunk patterns, p6spy verification |
| **Second-Level Caching** | Entity/collection/query cache, concurrency strategies, Ehcache/Caffeine/Redis config, @NaturalIdCache, query cache invalidation gotchas |
| **Connection Pooling** | HikariCP tuning, pool sizing formula, leak detection, statement caching, metrics |
| **Inheritance Strategies** | SINGLE_TABLE (usually best), JOINED, TABLE_PER_CLASS (avoid), @MappedSuperclass, composition over inheritance |
| **Spring Data JPA** | `BaseJpaRepository`, `findAll`/`findById` anti-patterns, `persist` vs `save`, EXISTS optimization, Stream methods, QBE, Jakarta Data, derived queries, `@Query`, `@Modifying` pitfalls, custom fragments, Specification API, projections, auditing, bidirectional sync, OSIV |
| **Spring Transactions** | `@Transactional` boundaries, propagation, isolation, optimistic/pessimistic locking, `LazyConnectionDataSourceProxy`, read-write routing, `@TransactionalEventListener`, OSIV |
| **Testing** | Testcontainers + `@ServiceConnection`, `@DataJpaTest` correctness, query count assertions, cleanup strategies, `@Sql`, migration tests |
| **Schema Migrations** | Flyway, `ddl-auto=validate`, repeatable migrations, zero-downtime column changes, `CREATE INDEX CONCURRENTLY`, multi-tenant schemas |
| **Hibernate 6 Features** | @DynamicUpdate/Insert, @Immutable, @Subselect, @Filter, @SQLRestriction, @SoftDelete, @TenantId, Statistics, custom types |
| **Migration 5→6** | javax→jakarta, naming changes, type system, HQL parser, Criteria API, Spring Boot 3 alignment |

---

## Installation

### Option A: Install from GitHub

```bash
# Claude Code skill installation
claude skill install https://github.com/YOUR_USERNAME/hibernate-jpa-validator-skill
```

### Option B: Manual Install

Clone and copy into your Claude plugins directory:

```bash
git clone https://github.com/YOUR_USERNAME/hibernate-jpa-validator-skill.git
cp -r hibernate-jpa-validator-skill ~/.claude/plugins/hibernate-jpa-validator/
```

---

## Usage

Once installed, the skill activates automatically when Claude detects Hibernate/JPA code. You can also invoke it explicitly:

```
/hibernate-jpa-validator
```

### Example Prompts

**Review an entity:**
```
Review this Post entity for Hibernate best practices:
[paste entity code]
```

**Diagnose slow queries:**
```
This endpoint is slow. Here's the service code:
[paste code]
Help me find and fix any N+1 or fetch strategy issues.
```

**Identifier strategy advice:**
```
Should I use @GeneratedValue(strategy = AUTO) or something else for my PostgreSQL app?
```

**Migration help:**
```
I'm migrating from Spring Boot 2.7 to 3.2. What Hibernate 5→6 changes do I need to make?
```

---

## How It Works

The skill follows the **Hypersistence Optimizer philosophy**: run the full checklist, show the generated SQL, explain the underlying JDBC behavior, then show the fix.

### SKILL.md (Core Workflow)

The main entry point. Routes requests to the right section:
- **Section B** — Entity Mapping Validation (always runs the full checklist)
- **Sections C–K** — Specific areas: N+1, queries, batching, caching, pooling, Spring Data, transactions, testing, schema migrations

### Reference Files

Deep-dive documentation with Java code examples (before/after patterns):

```
references/
├── entity-mapping-checklist.md    # Full entity review rules
├── identifier-strategies.md       # SEQUENCE, UUID, composite keys
├── association-mappings.md        # All 4 relationship types
├── fetching-and-n-plus-one.md     # N+1 detection and fixes
├── query-optimization.md          # Projections, keyset pagination, JOIN FETCH + pagination
├── batch-processing.md            # JDBC batching, flush/clear, bulk DML
├── caching.md                     # L2 cache strategies and gotchas
├── connection-pooling.md          # HikariCP tuning
├── inheritance-strategies.md      # SINGLE_TABLE, JOINED, etc.
├── spring-data-jpa.md             # BaseJpaRepository, anti-patterns, projections, QBE
├── spring-transactions.md         # @Transactional, locking, routing, OSIV
├── spring-testing.md              # Testcontainers, @DataJpaTest, query count assertions
├── spring-schema-migrations.md    # Flyway, ddl-auto=validate, zero-downtime patterns
├── logging-and-monitoring.md      # datasource-proxy, p6spy, Hibernate Statistics
├── hibernate-features.md          # Hibernate 6 specific features
└── migration-5-to-6.md            # Upgrade guide
```

---

## Inspiration and Credit

This skill is inspired by **Vlad Mihalcea's** work:

- [Hypersistence Optimizer](https://vladmihalcea.com/hypersistence-optimizer/) — automated JPA/Hibernate issue detector
- [High-Performance Java Persistence](https://vladmihalcea.com/books/high-performance-java-persistence/) — the definitive book on JPA performance
- [vladmihalcea.com](https://vladmihalcea.com) — blog with hundreds of detailed articles on Hibernate internals

The core insight from Vlad's work that this skill encodes: **understanding the underlying SQL and JDBC behavior is essential to writing correct, performant JPA code**. Hibernate is not magic — it translates your mappings into SQL, and knowing that SQL is the key to avoiding N+1, lock contention, and cartesian product explosions.

---

## Target Stack

- **Hibernate 6.x** (Spring Boot 3.x)
- **Spring Data JPA 3.x**
- **Java 17+**
- **PostgreSQL** (examples use PostgreSQL syntax; concepts apply to MySQL/MariaDB/Oracle)

Hibernate 5 / Spring Boot 2.x also supported with notes where behavior differs.

---

## Contributing

PRs welcome. If you find a pattern not covered, open an issue with:
1. The problematic code
2. The SQL Hibernate actually generates
3. The correct pattern and why it's better
