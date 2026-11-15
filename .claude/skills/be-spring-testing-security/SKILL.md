---
name: be-spring-testing-security
description: >
  Java/Spring backend: use when a test calls secured endpoints or services, or tests who may
  access what — fake callers with jwt(), authentication() and @WithMockUser, authorities as the
  Keycloak converter produces them, the access matrix (401/403/no-access reasons), CSRF for a
  bearer API, method security, and the one integration test against a real Keycloak in Testcontainers.
---

# Testing application security

Spring Security test support (`spring-boot-starter-security-test`, test scope) fakes an
authenticated caller by populating the `SecurityContext`. Load `be-spring-testing-web` alongside it
for HTTP tests. OIDC login, opaque tokens and WebFlux are not used by this backend.

The backend is a stateless OAuth2 resource server with Keycloak (`be-oauth2-resource-server`):
bearer JWTs, no login form, no session, CSRF off. Not yet verified by a build in this repository.

## Two levels

| Level | Suffix | What it proves | Keycloak |
|---|---|---|---|
| access matrix | `*Test` for a `@WebMvcTest` slice with mocks; `*IT` for tests in the shared context (`AbstractIT`) — the rule of `be-testing-conventions` | every rule of the access matrix, per operation | faked: `authentication(...)`, `jwt()`, `@WithMockUser` |
| **one** Keycloak IT | `*IT` | the real token passes our decoder: issuer, JWKS/RS256, audience, `azp`, claims, converter | real, in Testcontainers |

The fakes **bypass the decoder and the converter**. The matrix is therefore tested with fakes
(fast, many rows), the converter is unit-tested, and one IT proves the real chain end to end.

## Faking the caller — authorities as the converter produces them

API delegates and services read `AppAuthentication.current()`; a plain `jwt()` or `@WithMockUser`
gives a different type, and that code then fails. Build the caller with the **real converter** and a
fixed access decision, so the authorities are exactly what production produces:

```java
public final class TestAuthentications {   // backend-service src/test (the only application with security)

    // same values as the properties of AbstractIT (be-testing-conventions)
    private static final AppSecurityProperties PROPS = new AppSecurityProperties(
        "opereta-spa", "personal_number", "opereta-user", "opereta-api", "/api/users/me");

    public static AppAuthentication as(String personalNumber, AccessDecision decision) {
        Jwt jwt = Jwt.withTokenValue("test-token").header("alg", "none")
            .subject("account-" + personalNumber)
            .claim("personal_number", personalNumber)
            .claim("resource_access", Map.of("opereta-api", Map.of("roles", List.of("opereta-user"))))
            .build();
        return new KeycloakJwtAuthenticationConverter(PROPS, (principal, sub) -> decision).convert(jwt);
    }

    public static AppAuthentication active(String personalNumber, long personId, String... roleCodes) {
        return as(personalNumber, new AccessDecision(AccessState.ACTIVE, personId, Set.of(roleCodes)));
    }

    public static AppAuthentication withoutAccess(String personalNumber, AccessState reason) {
        Long personId = reason == AccessState.UNPAIRED ? null : 1L;
        return as(personalNumber, new AccessDecision(reason, personId, Set.of()));
    }
}

// in a test (personal numbers from the reserved test range, never real ones)
assertThat(mvc.get().uri("/api/projects/7")
        .with(authentication(TestAuthentications.active("900001", 42L, Roles.PROJECT_MANAGER))))
    .hasStatusOk();
```

| Need | Use |
|---|---|
| any caller of code that reads `AppAuthentication` | `.with(authentication(TestAuthentications.active(...)))` |
| unpaired, no role, suspended | `.with(authentication(TestAuthentications.withoutAccess(pn, AccessState.SUSPENDED)))` |
| a pure URL/role rule, code reads nothing | `.with(jwt().authorities(new SimpleGrantedAuthority("ROLE_ENTRY")))` — authorities exactly as above (`ROLE_ENTRY`, `ROLE_ACTIVE`, `ROLE_<CODE>`) |
| `@PreAuthorize` on a bean called directly | `@WithMockUser(roles = {"ACTIVE", "APP_ADMIN"})` |
| no caller | no annotation, no post-processor |

- `.with(...)` overrides a class- or method-level annotation for that request. Annotations act on the
  test thread only; a `RANDOM_PORT` server needs a real token (the Keycloak IT).
- Never sign own JWTs with a test key, never mock `JwtDecoder` to return arbitrary tokens in matrix
  tests — that tests the mock. Real signatures belong to the Keycloak IT.

## The access matrix

For each protected operation (read the filter chain and every `@PreAuthorize`), one test per row of
the general matrix in `be-testing-conventions`, made concrete here: `401` comes with
`WWW-Authenticate: Bearer error="invalid_token"` and an empty body; a token without the entry role
gets `403` everywhere; `ACTIVE` without the required business role gets `403`
`/problems/access-denied`.

**Opereta rows** (per ADR-00004 points 4–6):

- unpaired → `/api/users/me` `403` `application/problem+json`, `type` `/problems/no-access`,
  `reason: UNPAIRED`; every other endpoint the same `403` with `reason`; an audit record with the account's `sub`;
- paired and active → `/api/users/me` `200` with identity, roles, `permissions`, `Cache-Control: no-store`;
- a token with only the entry role → `/api/users/me` is reached (not caught by the `/api/**` rule) —
  guards that `app.security.me-path` equals the contract's path;
- suspended → `/api/users/me` **and any other** endpoint `403` `/problems/no-access`, `reason: SUSPENDED`;
- paired without any role → `reason: NO_ROLE`;
- a ban or a removed role written to the database → effective on the next request (or after the
  configured cache period, at most 5 minutes), without a new token — tested through `ApplicationAccess`
  against the database;
- token without `personal_number` → `401`;
- role administration: changing one's own roles → rejected; an administrator changing another
  administrator's roles → rejected; every accepted change audited.

## Unit-testing the converter

Plain JUnit, no Spring: `new KeycloakJwtAuthenticationConverter(props, stubAccess).convert(jwt)` with
a hand-built `Jwt`. Cover: principal claim missing or blank → `InvalidBearerTokenException`; entry role
as realm role and as client role of the configured client; entry role of **another** client ignored;
role names normalised (`opereta-user` → `OPERETA_USER`); business roles only when `ACTIVE`; token roles
other than the entry role never become authorities; `noAccessReason()` per state.

`ApplicationAccess` itself is a database read: test it as `*IT` against PostgreSQL (person paired,
unpaired, suspended, without role; role removed).

## Slices: load the real security configuration

A slice without our `SecurityFilterChain` runs under Boot's defaults and passes for the wrong reasons.

```java
@WebMvcTest(UsersApiController.class)                                  // generated controller of the area
@Import({UsersApiDelegateImpl.class, LibSecurityConfiguration.class})   // our delegate + real chain (+ lib-web)
class UsersApiSecurityTest {                                            // slice with mocks = *Test
    @MockitoBean CurrentUserQuery currentUserQuery;
    @MockitoBean ApplicationAccess applicationAccess;   // required by the converter bean, not called by fakes
    @MockitoBean JwtDecoder jwtDecoder;                 // no issuer in a slice; fakes never reach it
    @Autowired MockMvcTester mvc;
    …
}
```

`@MockitoBean` (Boot 4), never `@MockBean`. Every distinct set of mocks is a new context — keep one
slice per API area, or test the matrix in the shared context (`AbstractIT`, `be-testing-conventions`).

## CSRF

The API has CSRF disabled (bearer only). Assert once that a `POST` with a valid caller and **no** CSRF
token succeeds — it guards against someone switching CSRF back on. The cookie-based techniques
(`csrf()`, `MockMvcBuilderCustomizer`) apply only if the app ever authenticates with a cookie.

## Method security

Call the bean directly with the security configuration in the context:

```java
@Test
@WithMockUser(roles = {"ACTIVE", "EMPLOYEE"})
void approve_asEmployee_isDenied() {
    assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(() -> service.approve(7L));
}

@Test
void approve_withoutCaller_requiresAuthentication() {
    assertThatExceptionOfType(AuthenticationCredentialsNotFoundException.class)
        .isThrownBy(() -> service.approve(7L));
}
```

## One integration test against a real Keycloak

Proves what fakes cannot: the realm in the repository issues RS256 tokens our decoder accepts —
issuer, JWKS, audience, `azp`, `personal_number`, entry role. Exactly **one** class, cheap by design:
containers started once per JVM, one Spring context, a handful of requests.

```xml
<dependency>   <!-- version: testcontainers-keycloak.version in the parent POM (be-maven-reactor) -->
    <groupId>com.github.dasniko</groupId>
    <artifactId>testcontainers-keycloak</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@Tag("keycloak")
class KeycloakSecurityIT extends AbstractIT {   // shared IT base class (be-testing-conventions), MockMvc included

    static final Network NETWORK = Network.newNetwork();

    // TestImages holds the references of docker-compose.yml (tag@digest), one place in test code
    static final KeycloakContainer KEYCLOAK = new KeycloakContainer(TestImages.KEYCLOAK)
        .withContextPath("/auth").withNetwork(NETWORK).withNetworkAliases("keycloak");

    // the repo realm, applied exactly as docker-compose does (keycloak-config-cli, same image)
    static final GenericContainer<?> REALM = new GenericContainer<>(TestImages.KEYCLOAK_CONFIG_CLI)
        .withNetwork(NETWORK)
        .withCopyFileToContainer(MountableFile.forHostPath(repoPath("keycloak/config")), "/config")
        .withCopyFileToContainer(MountableFile.forClasspathResource("keycloak/it-client.yaml"), "/it/it-client.yaml")
        .withEnv(Map.of(
            "KEYCLOAK_URL", "http://keycloak:8080/auth",
            "KEYCLOAK_USER", KEYCLOAK.getAdminUsername(),
            "KEYCLOAK_PASSWORD", KEYCLOAK.getAdminPassword(),
            "IMPORT_FILES_LOCATIONS", "/config/realm.yaml,/config/development/*.yaml,/it/it-client.yaml",
            "IMPORT_VARSUBSTITUTION_ENABLED", "true",
            "KEYCLOAK_REDIRECT_URIS", "http://localhost/*",
            "APP_ADMIN_EMAIL", "it-admin@example.test",
            "APP_ADMIN_PASSWORD", "it-only-" + UUID.randomUUID()))   // + every other env lookup the realm still has
        .withStartupCheckStrategy(new OneShotStartupCheckStrategy().withTimeout(Duration.ofMinutes(3)));

    static {
        KEYCLOAK.start();
        REALM.start();   // runs once, must exit 0
    }

    @DynamicPropertySource
    static void oidc(DynamicPropertyRegistry registry) {
        String issuer = KEYCLOAK.getAuthServerUrl() + "/realms/app";   // the address the test fetches tokens from
        registry.add("spring.security.oauth2.resourceserver.jwt.issuer-uri", () -> issuer);
        registry.add("spring.security.oauth2.resourceserver.jwt.jwk-set-uri",
            () -> issuer + "/protocol/openid-connect/certs");
        registry.add("spring.security.oauth2.resourceserver.jwt.audiences", () -> "opereta-api");
        registry.add("app.security.client-id", () -> "it-direct-grant");
    }

    @Autowired MockMvcTester mvc;

    @Test
    void me_withRealToken_ofPairedPerson_returns200() {
        // insert the fictitious person whose personal number the test account carries
        assertThat(mvc.get().uri("/api/users/me")
                .header(AUTHORIZATION, "Bearer " + token(TEST_ACCOUNT)))
            .hasStatusOk();
    }

    @Test
    void me_withoutToken_returns401() { … }

    @Test
    void adminOperation_withEmployeeToken_returns403() { … }

    @Test
    void me_withTamperedToken_returns401() { … }   // flip one character of the signature

    static String token(String alias) {   // direct grant against the IT client
        var form = new LinkedMultiValueMap<String, String>();
        form.add("grant_type", "password");
        form.add("client_id", "it-direct-grant");
        form.add("username", alias);
        form.add("password", TestAccounts.password(alias));
        form.add("scope", "openid");
        Map<?, ?> body = RestClient.create().post()
            .uri(KEYCLOAK.getAuthServerUrl() + "/realms/app/protocol/openid-connect/token")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED).body(form)
            .retrieve().body(Map.class);
        return (String) body.get("access_token");
    }
}
```

- **Realm from the repository**, not a copy: `repoPath(...)` walks up from the working directory to
  the repository root. The same realm files as LOCAL, so a broken realm breaks this test.
- **Test-only client** `src/test/resources/keycloak/it-client.yaml`: a client with direct access grants
  and the API audience mapper, existing only in this test. The frontend client keeps only the
  authorization-code flow (per ADR-00004, "Jak se dodržení hlídá"); the IT therefore sets
  `app.security.client-id` to the IT client.
- **Test account by alias** from `keycloak/config/development/users.yaml`; `TestAccounts` reads its
  password from that file (SnakeYAML is on the test classpath). Never copy a password into test code.
  The account needs the `personal_number` attribute and the entry role (realm tasks in
  `be-oauth2-resource-server`).
- **Cost**: `@DynamicPropertySource` makes this one extra Spring context — acceptable for one class,
  never repeat it. Static containers start once and are removed by Testcontainers at JVM exit.
- **Opt-out locally**: tag `keycloak`, skipped with `-Dit.excludedGroups=keycloak` (Failsafe setup in
  `be-maven-reactor`); CI runs it. Docker is required (as for every `*IT`).
- `app.security.role-client-id` and the other OIDC values come from the properties of `AbstractIT`;
  this class overrides only issuer, JWKS URI, audience and client id.
- Not yet verified by a build in this repository (container start, `withContextPath`, issuer
  equality between the token and `getAuthServerUrl()`).

## Gotchas

- Agent uses plain `jwt()` for code that reads `AppAuthentication` — `authentication(TestAuthentications…)`
- Agent invents authorities (`ROLE_USER`, `SCOPE_read`) — use what the converter produces: `ROLE_ENTRY`, `ROLE_ACTIVE`, `ROLE_<CODE>`
- Agent tests only the happy path — every operation gets its 401, 403 and allowed rows
- Agent disables filters or adds a permissive test security configuration — always the real `LibSecurityConfiguration`
- Agent adds a second Keycloak IT class or a Keycloak container per test — one class, static containers
- Agent copies the realm into `src/test/resources` — import the repository realm; only the IT client is test-only
- Agent enables direct grants on the frontend client for the IT — use the test-only client
- Agent expects a redirect to a login page — a resource server answers `401`

## Provenance

Modified for this application's Java base (stateless OAuth2 resource server with Keycloak, Spring Security 7) — this file has been changed from the original.
Source: [Kehrlann/spring-boot-testing-skills](https://github.com/Kehrlann/spring-boot-testing-skills),
`skills/spring-testing-security`, Apache License 2.0 (see `LICENSE.txt`); original content by Daniel Garnier-Moiroux,
distilled from the book *Spring Boot Testing In Action*.
Changes: callers built with the real converter, access matrix with no-access reasons, slices with the
library configuration, one Keycloak integration test in Testcontainers; form-login, login-page and
HTML-escaping sections removed (not used by the base backend).
Opereta layer: access-matrix rows for paired accounts, suspension and role administration.
