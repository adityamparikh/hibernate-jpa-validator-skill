# Testing the Data Access Layer

Test the actual database, not a mock. Schemas, dialects, and SQL generation are where bugs live.

---

## Testcontainers — Real Database in Tests

Embedded H2/HSQLDB diverges from PostgreSQL/MySQL on constraints, JSON types, sequence behavior, and dialect. **Use Testcontainers.** Dependencies: `org.testcontainers:postgresql` + `org.testcontainers:junit-jupiter` (test scope).

### Spring Boot 3.1+: @ServiceConnection

```java
@SpringBootTest
@Testcontainers
class PostServiceIT {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
}
```

`@ServiceConnection` auto-wires the container into Spring Boot's datasource config — no manual `@DynamicPropertySource` boilerplate.

### Reusable Container (faster across test classes)

```java
public abstract class AbstractIT {

    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true);

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        r.add("spring.datasource.username", POSTGRES::getUsername);
        r.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

Enable reuse globally: `~/.testcontainers.properties` → `testcontainers.reuse.enable=true`.

---

## @DataJpaTest — Slice Test for Repositories

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = Replace.NONE)  // critical — don't replace with H2
class PostRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired
    PostRepository postRepository;

    @Autowired
    TestEntityManager em;

    @Test
    void findsBySlug() {
        em.persist(new Post("hello-world", "Hello"));
        em.flush();
        em.clear();  // ← essential — forces actual DB read, not L1 cache hit

        Optional<Post> found = postRepository.findBySlug("hello-world");
        assertThat(found).isPresent();
    }
}
```

**Two critical patterns:**
1. `@AutoConfigureTestDatabase(replace = Replace.NONE)` — without this, Spring substitutes H2 even when a real DB is configured.
2. `em.flush() + em.clear()` after setup — without it, your assertion reads from L1 cache, not the DB. False positives hide broken mappings.

---

## Asserting Query Count (N+1 Regression Tests)

```java
@SpringBootTest
@Testcontainers
class PostServiceQueryCountTest extends AbstractIT {

    @Autowired
    PostService postService;

    @Test
    void getActivePosts_issuesExactlyOneQuery() {
        // Arrange — seed data
        postService.createMany(20);

        // Act
        QueryCountHolder.clear();
        postService.getActivePosts();

        // Assert
        QueryCount count = QueryCountHolder.getGrandTotal();
        assertThat(count.getSelect())
            .as("Expected single query — N+1 regression?")
            .isEqualTo(1);
    }
}
```

Requires `datasource-proxy` configured in your test profile — see `references/logging-and-monitoring.md`.

This is the **most valuable single test** for performance regression. CI fails the moment someone adds a lazy access in a loop.

---

## Test Data Cleanup Strategies

| Strategy | Speed | Isolation | When to Use |
|---|---|---|---|
| `@Transactional` rollback | Fastest | Per-test | `@DataJpaTest`, simple cases |
| Truncate all tables | Fast | Per-test | Tests that commit (e.g., `@Async`) |
| Container per class | Medium | Per-class | Heavy schema, cross-test data |
| Container per test | Slow | Per-test | Last resort |

### Pattern 1: @Transactional Rollback (default in @DataJpaTest)

Spring rolls back the transaction at the end of each test. **Caveat:** does not work for code that calls `@Transactional(propagation = REQUIRES_NEW)` or commits explicitly.

### Pattern 2: Truncate Between Tests

```java
@SpringBootTest
@Testcontainers
class IntegrationTest extends AbstractIT {

    @Autowired
    JdbcTemplate jdbc;

    @AfterEach
    void cleanup() {
        jdbc.execute("""
            TRUNCATE TABLE post, author, comment RESTART IDENTITY CASCADE
            """);
    }
}
```

Faster than dropping/recreating the schema. `RESTART IDENTITY` resets sequences so test IDs are deterministic.

### Pattern 3: Schema Per Test (rare)

Use only when tests modify schema (DDL) — typically migration tests.

---

## Asserting Generated SQL

```java
@Test
void update_doesNotTouchUnchangedColumns() {
    // Capture SQL via Hibernate event listener or datasource-proxy
    QueryCountHolder.clear();
    postService.updateTitle(postId, "New Title");
    QueryCount count = QueryCountHolder.getGrandTotal();
    assertThat(count.getUpdate()).isEqualTo(1);
}

// For exact SQL inspection:
@Test
void dynamicUpdate_onlyDirtyColumns() {
    StatementInspector inspector = new StatementInspector();
    // Hook via hibernate.session_factory.statement_inspector
    postService.updateTitle(postId, "New Title");
    assertThat(inspector.lastStatement())
        .containsIgnoringCase("UPDATE post SET title=?")
        .doesNotContain("created_at=?");
}
```

---

## Mocking the Repository — Don't

A `@Mock PostRepository` with `when(...).thenReturn(...)` verifies nothing about SQL generation, mapping, or constraints — tests pass while prod fails. Mock the repository only when testing pure business logic that happens to *call* a repository (and the repository is covered by integration tests elsewhere). Anything touching mapping, JPQL, or SQL: real database, every time.

---

## @Sql for Test Data

```java
@Test
@Sql("/test-data/posts.sql")
void findActiveReturnsSeededRows() {
    List<Post> active = postRepository.findByStatus("ACTIVE");
    assertThat(active).hasSize(3);
}

// Or per-method:
@Test
@Sql(scripts = "/seed.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
void scenario() { ... }
```

Prefer SQL files over `entityManager.persist()` chains for complex graphs — faster, more readable, exercises the actual schema.

---

## Test Profiles

```properties
# application-test.properties
spring.jpa.hibernate.ddl-auto=validate         # never `update` or `create-drop` in tests
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE

# Disable open-in-view to catch lazy-loading bugs in tests
spring.jpa.open-in-view=false
```

`ddl-auto=validate` ensures your migrations and entity mappings agree. If they don't, the application won't start — which is exactly what you want in CI.
