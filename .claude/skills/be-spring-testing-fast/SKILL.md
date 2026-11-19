---
name: be-spring-testing-fast
description: >
  Java/Spring backend: keep Spring Boot test suites fast by avoiding ApplicationContext cache thrashing. Use when Spring tests are slow, when a test class adds properties, profiles, @MockitoBean, @TestConfiguration, a slice or @DirtiesContext, or when reviewing how a Spring test suite is configured.
---

# Fast Spring test suites

The Spring TestContext framework caches every application context it starts, keyed by the test class's configuration. A test class with the same configuration as an earlier one gets the running context for free; a different configuration starts a new one, which costs seconds to tens of seconds plus memory. The cache holds 32 contexts by default and evicts the least recently used, so a suite with many configurations starts, evicts and restarts the same contexts over and over.

**The central tenet: avoid context thrashing.** Aim for one shared context. Add a second (say, one "custom configuration" context next to the default) only for a behaviour that genuinely needs it, and know why it exists.

## What changes the cache key

Reading from the context never creates a new one: `@Autowired`, `@Value`, `@Qualifier`, injected `ApplicationContext`.

Each of the following makes a test class's configuration differ from every class that doesn't declare it *identically*:

- **Properties**: `properties = …` on `@SpringBootTest` or any slice, and `@TestPropertySource`. The key is the literal declaration. A text block vs a string array, a different property order, `key=value` vs `key = value`, the same values via `@SpringBootTest(properties)` vs `@TestPropertySource(properties)`: each counts as a different configuration.
- **Profiles**: `@ActiveProfiles`.
- **Configuration classes**: `@SpringBootTest(classes = …)`, any `@TestConfiguration` (nested or `@Import`-ed, even an empty nested one), `@Import`.
- **Auto-configuration**: `@AutoConfigureMockMvc`, `@AutoConfigureRestTestClient` and the other `@AutoConfigure…` annotations, when some classes have them and others don't.
- **Bean overrides**: `@MockitoBean`, `@MockitoSpyBean`, `@TestBean`. The set of overridden beans is part of the key.
- **Slices**: each slice annotation, and each distinct parameter (`controllers = …`, `components = …`).
- **Web environment**: `MOCK` vs `RANDOM_PORT` vs `NONE`.
- **`@DynamicPropertySource`**: the declaring methods are part of the key, so each class that declares its own gets its own context. Register values once through a `DynamicPropertyRegistrar` bean in the shared configuration instead.
- **`@DirtiesContext`**: evicts the context, forcing a rebuild for the next class that needs it.
- Context initializers, loaders, parent context hierarchies.

## Steps: auditing a suite

1. **Measure.** Run the suite with the `org.springframework.test.context.cache` logger at `DEBUG`: pass `-Dlogging.level.org.springframework.test.context.cache=DEBUG` to the test JVM (Maven forwards `-D` user properties to Surefire; Gradle needs `systemProperty` in the `test` task), or add the logger to a `logback-test.xml` in `backend/backend-service/src/test/resources/`. Read the `Spring test ApplicationContext cache statistics` lines: `size` and `missCount` count contexts started. `hitCount` grows faster than the number of tests and needs no attention. `TRACE` prints each context's key, which shows exactly what differs.
2. **Inventory configurations.** For every Spring test class, list its annotations from the cache-key list, for example by grepping for `@\w+Test\b|@AutoConfigure|@MockitoBean|@MockitoSpyBean|@TestBean|@TestPropertySource|@ActiveProfiles|@TestConfiguration|@Import|@DynamicPropertySource|@DirtiesContext`. Group classes that end up with identical configurations.
3. **Converge incidental differences.** For each group that differs only by accident:
   - Move shared mocks, properties and auto-configuration into the application's shared IT base class (below), and extend it everywhere.
   - Mock the same set of beans in every class, even if a given class only stubs one of them.
   - Declare each property set once, in one literal.
   - Fold one-off `@TestConfiguration` beans into the shared configuration.
   - Replace slices that save little startup time with the shared `@SpringBootTest`.
   - Replace `@DirtiesContext` with a state reset (see **Pollution**).
   - Move tests that vary configuration only to check binding or validation into unit tests or `SpringApplicationBuilder` tests (load `be-spring-testing-properties`).
4. **Re-measure.** Done when `missCount` equals the number of configurations you decided to keep, each with a stated reason.

## The shared IT base class

Put every customization the suite needs on one abstract base class per application, so all test classes share one cache key. The names are fixed by `be-testing-conventions`: `AbstractIT` (`backend-service`), `AbstractSchedulerIT` (`scheduler-service`); every subclass is an `*IT`.

```java
@SpringBootTest(properties = { "startup.delay=100ms" })
@AutoConfigureMockMvc
@Import(TestcontainersConfiguration.class)
@MockitoBean(types = { WeatherService.class, Gizmo.class })
public abstract class AbstractIT {
}
```

Test classes `extends AbstractIT`, then `@Autowired` the mocked types to stub them. Mockito beans reset after each test by default, so stubbing in one test does not leak into the next. A subclass adds no annotation from the list above.

## Pollution: the price of sharing

A shared context means shared state: database rows, caches, in-memory singletons. To keep tests order-independent:

- **Reset the universe** in `@BeforeEach`, for example `repository.deleteAll()`. Alternatively, make each test's data unique (a random user, a fresh id) and assert only on that data.
- Keep tests independent of execution order; ordering tests to make them pass couples them.
- Use `@DirtiesContext` (after the method by default; `methodMode = BEFORE_METHOD` for the opposite) only for a test that breaks the context beyond repair. It rebuilds the app, and it does not clean anything outside the app, such as a database in a container.

## Other levers

- **Slices pay off only for large apps.** Many thin slices, each its own context, run slower than the same tests against one shared context. Load `be-spring-testing-strategy` for when a slice is worth it.
- **The cache is a static field, one per JVM.** Build-tool forks (Surefire `forkCount > 1`, `reuseForks=false`) each rebuild their contexts. Parallel tests inside one JVM share contexts *and their state*: run them in parallel only when every test is isolated from the others.
- **One database container per JVM.** A deliberate second context need not start a second container: the `@ServiceConnection` bean returns a static container (`TestcontainersConfiguration`), started by the first context; the next one finds the schema migrated, and the row reset keeps tests independent. Closing a context stops it, one more reason for no `@DirtiesContext`.
- **Overlap slow infrastructure (target, not built in this repository).** A container only one class needs and that boots slowly (Keycloak) could start in the background when the test plan begins (a `TestExecutionListener` in `META-INF/services`), and that class would run last (`@Order` plus `ClassOrderer$OrderAnnotation` in `junit-platform.properties`), so the other classes run while it boots. Today `KeycloakSecurityIT` starts its container in a static block (`be-spring-testing-security`).
- **Raising the cache limit hides the problem.** Every cached context holds threads, connection pools and possibly containers; small CI machines run out of memory first. Fix the number of configurations.
- **`SpringApplicationBuilder` tests bypass the cache.** Keep those contexts to a handful of configuration classes so each costs tens of milliseconds.

## Provenance

Modified for this application's Java base — this file has been changed from the original.
Source: [Kehrlann/spring-boot-testing-skills](https://github.com/Kehrlann/spring-boot-testing-skills),
Apache License 2.0 (see `LICENSE.txt`); original content by Daniel Garnier-Moiroux.
Changes: the shared meta-annotation replaced by the shared IT base class of `be-testing-conventions`.
