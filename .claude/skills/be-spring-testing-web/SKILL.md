---
name: be-spring-testing-web
description: >
  Java/Spring backend: HTTP-level tests for Spring Boot web apps with MockMvcTester, RestTestClient or WebTestClient - requests, JSON assertions, stubbing collaborators, full-server RANDOM_PORT tests. Use when testing controllers, REST/JSON endpoints, error handling and status codes, or WebFlux handlers.
---

# Testing Spring web applications

Default stack (Spring Boot 4 / Spring Framework 7, Spring MVC): `@SpringBootTest` + `@AutoConfigureMockMvc` + `MockMvcTester` with AssertJ. The test dependency is `org.springframework.boot:spring-boot-starter-webmvc-test`, in `test` scope.

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.assertj.MockMvcTester;
import static org.assertj.core.api.Assertions.assertThat;
```

Other cases:

- **WebFlux apps**, legacy `MockMvc` / `TestRestTemplate`, or `WebTestClient` / `RestTestClient` style: read [other-clients.md](other-clients.md).
- **HTML structure, forms, clicks, JavaScript**: not served by this backend (the frontend has its own tests).
- **Secured endpoints** (401/403, CSRF, fake users): load `be-spring-testing-security` alongside this skill.

## Steps: writing a web test class

1. **Extend the application's shared IT base class** (`AbstractIT`, `be-testing-conventions`); the class is an `*IT`. Any extra per-class configuration costs a new context; load `be-spring-testing-fast` first.
2. **Stub external systems.** The beans that reach outside the app (e.g. the mail sender) are `@MockitoBean`s declared once in the base class; autowire them, give them default answers in `@BeforeEach`, and override per test where the scenario needs it.
3. **Reset state** in `@BeforeEach` (`repository.deleteAll()`), because the context and its data are shared across test classes.
4. **Add domain helpers** for repeated arrange steps (`selectCity("Paris")`), so each test reads as intent.
5. **Write each test as arrange / act / assert.** Keep the exchange result in a `response` variable, then assert.
6. **Cover the error paths**, not just the happy path: invalid fields (422), unreadable body (400), not found (404), conflicts (409), and whatever the global handler maps (`be-problem-details-rfc9457`). Use `doThrow(...).when(mock).method(...)` for `void` methods.

Done when every endpoint you touched has at least its success status, its body contract and each mapped error status asserted.

```java
class WeatherApiIT extends AbstractIT {   // @SpringBootTest + @AutoConfigureMockMvc + mocks live in the base class

    @Autowired MockMvcTester mvc;
    @Autowired SelectionRepository selectionRepository;
    @Autowired WeatherService weatherService;   // a @MockitoBean declared in AbstractIT

    @BeforeEach
    void setUp() {
        selectionRepository.deleteAll();
        when(weatherService.getCurrentWeather(anyDouble(), anyDouble()))
            .thenReturn(new WeatherData(20, 0, 0));
    }

    @Test
    void getWeather() {
        var paris = selectCity("Paris");

        var response = mvc.get().uri("/api/weather").exchange();

        assertThat(response)
            .hasStatus(HttpStatus.OK)
            .bodyJson()
            .isLenientlyEqualTo("""
                [ { "cityName": "Paris", "country": "France", "temperature": 20.0 } ]
                """)
            .extractingPath("$.[0].cityId").isEqualTo(paris.getId());
    }
}
```

## MockMvcTester reference

**Requests**: `mvc.get() | post() | put() | delete()`, then:

- `.uri("/api/city/{id}", id)`: always a path, never a host or port. There is no server.
- `.queryParam(k, v)`, `.header(k, v)`, `.cookie(c)`, `.accept(type)`, `.apiVersion(v)`.
- `.formField(name, value)` submits an HTML form.
- `.content(stringOrBytes)` sends the body as-is: also set `.contentType(MediaType.APPLICATION_JSON)`.
- `.with(postProcessor)` applies a `RequestPostProcessor`. Wrap headers or cookies repeated on many requests in one, for example `private static RequestPostProcessor apiUser(String name) { return request -> { request.addHeader(...); request.setCookies(...); return request; }; }`.
- `.exchange()` performs the request. `.exchange().assertThat()…` chains straight into assertions.

**Assertions** on `assertThat(response)`:

- Status and headers: `.hasStatus(HttpStatus.CREATED)`, `.hasStatus2xxSuccessful()`, `.hasRedirectedUrl("/login")`, `.hasErrorMessage(...)`, `.body().isEmpty()`.
- `.bodyText().contains(...)`: fine for a sanity check on text. On failure it prints the whole body.
- JSON:
  - `.bodyJson().isLenientlyEqualTo(json)` ignores property order and properties missing from the expected document. Prefer it: assert the stable fields, skip generated ids.
  - `.bodyJson().isEqualTo(json)` is strict.
  - `.bodyJson().extractingPath("$.[0].cityId").isEqualTo(id)` checks a JsonPath; `extractingPath("$.[*].name").asArray().containsOnly(...)`, `extractingPath("$.length()")`.
  - `.bodyJson().convertTo(InstanceOfAssertFactories.list(WeatherResponse.class))` converts through the app's message converters, so you can use regular AssertJ on objects. Declare a test record with `@JsonIgnoreProperties(ignoreUnknown = true)` holding only the fields you check.
- Ignore specific values in a strict comparison: `.bodyJson().isEqualTo(expected, JsonAssert.comparator(new CustomComparator(JSONCompareMode.LENIENT, new Customization("[*].cityId", (a, e) -> true))))`.
- `MockMvcTester` deserializes JSON only. For XML or other formats, read `response.getResponse().getContentAsByteArray()` and parse it yourself.

## Web slice: `@WebMvcTest`

Only for a large, slow-starting app where you want a fast loop on the web layer; otherwise use the shared `@SpringBootTest` (load `be-spring-testing-strategy` for the trade-off).

- `@WebMvcTest(controllers = { A.class, B.class })` (`org.springframework.boot.webmvc.test.autoconfigure`) loads those controllers, `@ControllerAdvice`, converters, filters and other web components, and auto-configures `MockMvcTester`. Services and repositories are not loaded. With the delegate pattern the controllers are the generated `…ApiController`s; `@Import` the matching `…ApiDelegateImpl` (and `LibSecurityConfiguration`). A slice with mocks is a `*Test` (`be-testing-conventions`).
- Provide the controllers' dependencies with `@MockitoBean`.
- Group related controllers that share dependencies in one slice. A slice per controller is one context per controller, which thrashes the cache.
- A slice doesn't load your `SecurityFilterChain`: import it (load `be-spring-testing-security`).
- Everything else in this skill (requests, assertions, the mock MVC limits below) applies unchanged.

## What the mock MVC stack does not run

`webEnvironment = MOCK` (the default) calls the `DispatcherServlet` directly from the test thread. No servlet container starts. Consequences:

- **No container features**: error pages (`/error/404.html` comes back with an empty body), container-managed sessions, Tomcat valves. Spring's own error message is still available through `.hasErrorMessage(...)`.
- **Controllers run on the test thread**: thread-locals such as the `SecurityContext` or a test-level `@Transactional` reach the controller.
- **Requires Spring MVC**: WebFlux apps use `WebTestClient` (see [other-clients.md](other-clients.md)).

## Full-server tests

When you need what the container does, start the real server on a random port:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureRestTestClient // org.springframework.boot.resttestclient.autoconfigure
class ErrorPageIT {   // its own context (RANDOM_PORT): only where the container itself is under test

    @Autowired RestTestClient client; // base URL preset to the running server

    @Test
    void displaysCustom404Page() {
        var spec = client.get().uri("/does-not-exist").accept(MediaType.TEXT_HTML).exchange();
        var response = RestTestClientResponse.from(spec); // AssertJ bridge

        assertThat(response).hasStatus(HttpStatus.NOT_FOUND)
            .bodyText().containsIgnoringCase("404 NOT FOUND");
    }
}
```

- Use `RANDOM_PORT`. `DEFINED_PORT` collides with other processes and with other test contexts, so use it only when the port must be known before startup.
- `@LocalServerPort int port` gives the port to clients you build yourself, such as `RestClient.builder().baseUrl("http://localhost:" + port)`.
- **Debugging**: put a breakpoint in the test, read `port`, and open `http://localhost:<port>` in a browser to see the app with the test's data. In IntelliJ IDEA, set the breakpoint to suspend only the test thread (right-click › Suspend › Thread), otherwise the server is frozen too.
- Each `webEnvironment` value is a separate cached context: keep full-server tests to the ones that need the container.

## Provenance

Modified for this application's Java base — this file has been changed from the original.
Source: [Kehrlann/spring-boot-testing-skills](https://github.com/Kehrlann/spring-boot-testing-skills),
Apache License 2.0 (see `LICENSE.txt`); original content by Daniel Garnier-Moiroux.
Changes: test classes extend the shared IT base class and use the `*IT` suffix; error statuses per
ADR-00007 (422/400); slices with the generated controllers; browser-testing references removed.
