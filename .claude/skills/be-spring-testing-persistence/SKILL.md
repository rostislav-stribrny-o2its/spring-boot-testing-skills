---
name: spring-testing-persistence
description: Test the persistence layer of Spring Boot apps on Spring Data JDBC and JdbcClient - aggregate repositories, custom and JdbcClient queries, streaming reads, aggregate mappings, database constraints, Flyway migrations and transactions - with PostgreSQL in Testcontainers and @ServiceConnection, @DataJdbcTest/@JdbcTest and @Sql. Use when adding or changing repositories, queries, entities or migrations, or when Spring tests need a database.
---

# Testing persistence

Test what is built from *your* declarations: derived query method names, `@Query` SQL, `JdbcClient` queries and their row mappers, custom repository implementations, aggregate mappings (`@Table`, `@MappedCollection`, `@Embedded`, converters, `@Version`), the persistence mapper between domain and `…Entity`, database constraints, and migrations. Plain `save` / `findById` is Spring Data's own code; flow tests exercise it.

In the Elysium Java base persistence is Spring Data JDBC (aggregate writes) plus `JdbcClient` (reads) on PostgreSQL 18. There is no JPA: no persistence context, no lazy loading, no `TestEntityManager`.

## Steps

1. **Inventory** the repositories, `JdbcClient` queries and entities you touched: queries, mappings, constraints, migrations.
2. **Run against the production engine** with a Testcontainers database shared by the whole suite (below).
3. **Write repository tests in the suite's shared `@SpringBootTest`**: autowire the repositories, reset data in `@BeforeEach`. Use a data slice only in a large, slow-starting app (load `spring-testing-strategy`).
4. **Per query**: data that must match, and look-alike data that must not (boundary dates, case differences, nulls, another user's rows). Assert ordering and paging when the query declares them.
5. **Per mapping**: save, then reload from the database (see **Round trips**) and assert every mapped field, child collection and the incremented `@Version`.
6. **Per constraint**: violate it and expect `DataIntegrityViolationException`.

Done when every derived or custom query has a matching and a non-matching case, every mapping is asserted after a reload from the database, every constraint has a violation test, and the suite's context starts against the production engine with its migrations applied.

## One database container for the whole suite

Dependencies (test scope): `org.springframework.boot:spring-boot-testcontainers` and the Testcontainers module for the engine, such as `org.testcontainers:testcontainers-postgresql` (Testcontainers 2).

```java
import org.testcontainers.postgresql.PostgreSQLContainer;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer postgres() {
        return new PostgreSQLContainer("postgres:18"); // same major version as production
    }
}
```

- `@ServiceConnection` points the datasource at the container; no URL or credential properties needed. The container starts with the context and stops when the context closes.
- `@Import(TestcontainersConfiguration.class)` from the suite's shared meta-annotation, so every class shares one context and one container (load `spring-testing-fast`).
- Starting the context runs Flyway against the container, so `contextLoads()` proves the migrations apply to the real engine. The image's major version matches production (PostgreSQL 18); never test on H2 or another major version.
- **Tie the container's lifecycle to the context, as above.** A `static @Container` field under `@Testcontainers` is stopped by JUnit after its class, while the cached context still points at it: the next class that reuses that context fails with connection errors. Its `@DynamicPropertySource` method also gives each class its own cache key.
- To run the app locally against the same container, add a `main` in `src/test`: `SpringApplication.from(MyApplication::main).with(TestcontainersConfiguration.class).run(args)`.

## Repository tests

```java
@AppTest // the suite's shared meta-annotation, importing TestcontainersConfiguration
class CityRepositoryTests {

    @Autowired CityRepository cityRepository;

    @BeforeEach
    void setUp() {
        cityRepository.deleteAll();
    }

    @Test
    void findsByNamePrefixIgnoringCase() {
        cityRepository.saveAll(List.of(city("Paris"), city("Parma"), city("Lyon")));

        var cities = cityRepository.findByNameStartingWithIgnoreCase("par");

        assertThat(cities).extracting(City::getName).containsExactlyInAnyOrder("Paris", "Parma");
    }
}
```

- **Reset order**: with foreign keys, delete children before parents, or `JdbcTestUtils.deleteFromTables(jdbcTemplate, "selection", "city")` in child-to-parent order.
- **SQL fixtures**: `@Sql("/fixtures/cities.sql")` on the class or a method runs before each test; `executionPhase = AFTER_TEST_METHOD` for cleanup scripts. `@Sql` doesn't change the cache key.
- Repository tests assert on repositories; behaviour tests elsewhere assert through HTTP and services.

## Round trips

Spring Data JDBC has no persistence context: `save(...)` runs the SQL immediately and `findById(...)` always queries the database and builds a new object. A save-then-reload test therefore proves the mapping — as long as you assert on the **reloaded** object, not on the one you saved (that one only has the generated ID and version set).

- Assert the whole aggregate after reload: root columns, every `@MappedCollection` child (count, order for `List`, keys for `Map`), embedded values, converted types (`BigDecimal` scale, `Long` IDs, lookup-table FKs).
- Save the reloaded aggregate again and assert `@Version` incremented; save a stale copy and expect `OptimisticLockingFailureException`.
- With a separate domain model, test the round trip through the `@Repository` that applies the persistence mapper: domain → `…Entity` → database → `…Entity` → domain, and compare the domain objects.
- Constraints fire when the SQL runs, i.e. inside `save(...)`; expect `DataIntegrityViolationException` from that call.

## `JdbcClient` queries and streaming reads

- Test each read query with matching and non-matching rows, and assert the projected `…Row` fields (types, nulls, scale of `numeric`).
- A query returning a `Stream` must be consumed inside a transaction and closed: in the test, wrap the call in a `TransactionTemplate` and a try-with-resources block, the same way production code does. A test that reads the stream outside a transaction hides the real failure mode.
- To prove streaming really happens (PostgreSQL streams only with autocommit off and a fetch size), assert the configured fetch size on the `JdbcTemplate` behind the `JdbcClient` (`spring.jdbc.template.fetch-size`), or test the query against a data set larger than the fetch size.

## Transactions in tests

`@Transactional` on a test class or method rolls back after each test. Keep it to repository-level tests. In flow tests it changes what runs:

- With `MockMvcTester`, the controller runs on the test thread inside the test's transaction: a stream consumed after the service returned still works, and commit-time failures never happen.
- With `RANDOM_PORT` or `@Async` code, other threads can't see the test's uncommitted data, and the rollback doesn't undo what they wrote.

For flow tests, reset data in `@BeforeEach` instead. To test a `@Transactional` service's rollback, keep the test non-transactional: make a collaborator throw (`doThrow` on a `@MockitoBean`, or a `@MockitoSpyBean`), call the service, then assert the repository is unchanged.

## Slices: `@DataJdbcTest`, `@JdbcTest`

Only for a large, slow-starting app (load `spring-testing-strategy`). If the suite already has `@SpringBootTest` classes, autowire repositories there instead: you reuse the cached context and pay no second startup.

- `@DataJdbcTest` loads Spring Data JDBC repositories, the mapping context, the datasource and migrations; `@JdbcTest` loads the datasource, `JdbcTemplate`/`JdbcClient` and migrations, for read queries without repositories. Each comes from the matching Boot 4 test starter (Spring Data JDBC, or JDBC). `@DataJpaTest` does not apply — the base has no JPA.
- Your `@Repository` classes that wrap a Spring Data repository and a MapStruct persistence mapper are not component-scanned by these slices: `@Import` them (and the mapper) explicitly.
- Every test is `@Transactional` and rolled back. Opt out with `@Transactional(propagation = Propagation.NOT_SUPPORTED)` on the class or method.
- `@AutoConfigureTestDatabase` (default `replace = NON_TEST`) swaps the datasource for an embedded H2, HSQLDB or Derby *unless* it already points at a test database: a `@ServiceConnection` container, a `@DynamicPropertySource` URL, or a Testcontainers JDBC URL. `@Import(TestcontainersConfiguration.class)` therefore keeps the real engine. Without one, the embedded database must be on the test classpath.
- An embedded database can't run engine-specific SQL: native queries, and migrations that use the engine's types or functions.
- `properties = …` and each slice annotation change the cache key, as on `@SpringBootTest`.

## Migrations

Flyway CI checks (format and duplicate versions, `flyway validate`, empty database, upgrade from the last release) are described in the `flyway-migrations` skill; the persistence suite runs on the database those checks produce.

## Provenance

Modified for the Elysium Java base (Spring Data JDBC + `JdbcClient`, no JPA, PostgreSQL 18) — this file has been changed from the original.
Source: [Kehrlann/spring-boot-testing-skills](https://github.com/Kehrlann/spring-boot-testing-skills),
`skills/spring-testing-persistence`, Apache License 2.0 (see the repository `LICENSE`); original content by Daniel Garnier-Moiroux,
distilled from the book *Spring Boot Testing In Action*.
Changes: JPA-specific round trips and `@DataJpaTest` replaced by Spring Data JDBC and `JdbcClient`, streaming reads, PostgreSQL 18 image.
