# PROMPT — HEALTH ENDPOINTS + SPRING BOOT ADMIN
## For use against: notification-service, order-service, payment-service, product-service, telemetry-service, user-service + new admin-service

---

You are an expert Spring Boot production-readiness engineer. You will implement comprehensive health endpoints across all six existing microservices and create a new seventh service — `admin-service` — that aggregates them into a Spring Boot Admin dashboard.

This is an existing codebase. **Read before you write anything.**

## THE SIX EXISTING SERVICES

The working directory contains six Spring Boot + Gradle microservices:
- `notification-service`
- `order-service`
- `payment-service`
- `product-service`
- `telemetry-service`
- `user-service`

## THE NEW SERVICE TO CREATE

- `admin-service` — Spring Boot Admin Server, created from scratch in the working directory

## TECHNOLOGY STACK

- **Java**: 17+
- **Spring Boot**: 3.x (read the version from each service's `build.gradle` before proceeding)
- **Build tool**: Gradle (Groovy DSL) for all six existing services; use the same for `admin-service`
- **Spring Boot Admin**: `de.codecentric:spring-boot-admin-starter-server:3.3.5` (server) and `de.codecentric:spring-boot-admin-starter-client:3.3.5` (client)

---

## STEP 0 — CREATE BRANCHES (DO THIS FIRST, BEFORE ANY OTHER ACTION)

Before reading any file or writing any code, create the working branch in every existing repo:

```bash
# Run from the working directory root — one command per repo
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    cd $svc
    git checkout -b ai-single-prompt-health
    cd ..
done
```

Verify all six branches exist before continuing:

```bash
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: "
    cd $svc && git branch --show-current && cd ..
done
# Expected output: ai-single-prompt-health for every service
```

The `admin-service` branch is created after the directory is initialised in Step 5. All code written from Step 1 onwards goes onto `ai-single-prompt-health`. Never commit to `main`.

---

## STEP 1 — DISCOVERY (READ EVERYTHING BEFORE WRITING ANYTHING)

For **each** of the six existing services, read and record:

1. The full `build.gradle` — note the Spring Boot version, all existing dependencies (especially `spring-boot-starter-actuator` if already present, any database drivers, any cache dependencies, any messaging dependencies)
2. Every `src/main/resources/application.yml` and `application-*.yml` — record `spring.application.name`, `server.port`, and any existing `management.*` configuration
3. Every class that calls another service via `RestTemplate`, `WebClient`, `FeignClient`, or `RestClient` — list the service name and base URL so you can write a downstream health indicator for it
4. Any `DataSource`, `JdbcTemplate`, `JpaRepository`, or database-related configuration — presence means the DB health indicator must be explicitly configured and verified
5. Any other infrastructure dependencies: Redis, Kafka, RabbitMQ, external APIs, file system paths — each needs its own health indicator
6. Any Spring Security configuration — look for classes annotated `@EnableWebSecurity` or beans returning `SecurityFilterChain`. If found, record it: the actuator endpoints will return **403** to unauthenticated callers (including downstream health indicators from other services) unless explicitly permitted. See Step 4f.

Write a **health readiness map** in this exact format before touching any file:

```
HEALTH READINESS MAP
====================

[service-name]
  spring.application.name: <value>
  server.port: <value>
  spring-boot-starter-actuator already present: YES / NO
  Database (type): <H2 / PostgreSQL / MySQL / NONE FOUND>
  DataSource health indicator: AUTO (Spring provides it) / NEEDS CUSTOM / NONE
  Downstream services called:
    - <service-name> at <base URL> → needs DownstreamHealthIndicator
  Other infrastructure: <Redis / Kafka / RabbitMQ / NONE FOUND>
  Existing management.* config: <list or NONE FOUND>
```

**Do not proceed to Step 2 until the health readiness map is complete for all six services.**

---

## STEP 2 — DEPENDENCY ADDITIONS (ALL SIX EXISTING SERVICES)

For every service that does NOT already have `spring-boot-starter-actuator`, add it to `build.gradle` inside the existing `dependencies {}` block without altering any existing lines:

```groovy
// Actuator — add inside existing dependencies block only if not already present
implementation 'org.springframework.boot:spring-boot-starter-actuator'

// Spring Boot Admin client — registers this service with the admin-service dashboard
implementation 'de.codecentric:spring-boot-admin-starter-client:3.3.5'
```

Do NOT add `spring-boot-starter-actuator` if it is already listed — check the build file first.

Also add the `git-commit-id` Gradle plugin to **every** service's `build.gradle` `plugins {}` block. This plugin runs at build time and generates `src/main/resources/git.properties`, which Spring Boot reads to populate `/actuator/info` with branch name, commit hash, and build timestamp. Without this plugin the git info properties in `application.yml` have nothing to read and `/actuator/info` will show an empty `git` object.

```groovy
plugins {
    // ADD to existing plugins block — do not replace any existing entries
    id 'com.gorylenko.gradle-git-properties' version '2.4.1'
}
```

Configure the plugin to generate the full git metadata (add this block after the `dependencies {}` block, not inside it):

```groovy
gitProperties {
    // Keys to include in git.properties — 'full' mode in application.yml reads all of these
    keys = [
        'git.branch',
        'git.commit.id',
        'git.commit.id.abbrev',
        'git.commit.message.short',
        'git.commit.time',
        'git.build.time',
        'git.build.version',
        'git.dirty',
        'git.remote.origin.url',
        'git.tags'
    ]
    dotGitDirectory = project.file("${project.rootDir}/.git")
}
```

**WARNING:** Do NOT add `generateGitPropertiesFile` or `generateGitPropertiesFilename` to this block — those properties do not exist in plugin version 2.4.1 and will cause the build to fail immediately with `Could not set unknown property`. The plugin writes to the correct location (`build/resources/main/git.properties`) by default without any extra configuration.

---

## STEP 3 — ACTUATOR CONFIGURATION (ALL SIX EXISTING SERVICES)

Add or merge the following into each service's `application.yml`. **Merge carefully** — do not overwrite existing keys, do not duplicate keys that already exist. Use YAML block style.

```yaml
# Health endpoint configuration — merge into existing application.yml

spring:
  application:
    name: <READ FROM EXISTING YML — do not change this>

management:
  endpoints:
    web:
      exposure:
        # Expose health + info for Spring Boot Admin; restrict everything else
        include: health,info,metrics,loggers
  endpoint:
    health:
      show-details: always       # Spring Boot Admin reads detailed health; always show
      show-components: always
      probes:
        enabled: true            # Enables /actuator/health/liveness and /actuator/health/readiness
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  info:
    env:
      enabled: true              # Allows Spring Boot Admin to read /actuator/info
    git:
      enabled: true              # Exposes git metadata (branch, commit, timestamp) at /actuator/info
      mode: full                 # full = all fields from git.properties; simple = only commit.id and time

# Spring Boot Admin client registration
spring:
  boot:
    admin:
      client:
        url: http://localhost:8099    # admin-service runs on port 8099 — SAME value in every service
        instance:
          name: ${spring.application.name}
          prefer-ip: true

# Info endpoint metadata — visible in Spring Boot Admin UI
info:
  app:
    name: ${spring.application.name}
    description: <short one-line description matching the service name>
    version: 1.0.0
```

**Consistency rules — these values must be identical across all six services:**
- `management.endpoints.web.exposure.include` = `health,info,metrics,loggers`
- `management.endpoint.health.show-details` = `always`
- `spring.boot.admin.client.url` = `http://localhost:8099` (no trailing slash, identical string)
- `management.info.git.enabled` = `true`
- `management.info.git.mode` = `full`

**Do not change `spring.application.name`** — read whatever is there and leave it as-is.

---

## STEP 4 — HEALTH INDICATORS (ALL SIX EXISTING SERVICES)

### 4a — Built-in indicators (no code required — verify they are active)

Spring Boot automatically activates these indicators when the corresponding dependency is on the classpath:
- `DataSourceHealthIndicator` — active when a `DataSource` bean exists (H2, PostgreSQL, MySQL, etc.)
- `DiskSpaceHealthIndicator` — always active
- `PingHealthIndicator` — always active
- `RedisHealthIndicator` — active when `spring-boot-starter-data-redis` is present
- `RabbitHealthIndicator` — active when `spring-boot-starter-amqp` is present

For each service, confirm which built-in indicators will be active based on what you found in Step 1.

### 4b — Custom downstream health indicator

For every service that calls another service over HTTP (discovered in Step 1), create a custom `HealthIndicator` at:

`src/main/java/{base_package}/health/{TargetServiceName}HealthIndicator.java`

**Critical design rules for downstream health indicators:**

1. **Use a short timeout** — downstream checks must not block indefinitely. Set a 2-second connect timeout and 3-second read timeout. A slow downstream should report `DOWN` with a `responseTime` detail, not hang the health endpoint.
2. **Never throw uncaught exceptions** — wrap the HTTP call in try/catch. A connection refused, timeout, or 5xx response means `DOWN` with the error message as a detail. An uncaught exception causes the entire `/actuator/health` to return 500, which breaks Spring Boot Admin.
3. **Call `/actuator/health` on the downstream service**, not a business endpoint — health endpoints are purpose-built for this check.
4. **Include `responseTimeMs` in the health details** — it makes Spring Boot Admin useful beyond just UP/DOWN.
5. **Annotate the class as a component** using `@Component` — Spring discovers it automatically, no registration code needed.

```java
package com.example.orderservice.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.client.RestClientException;
import org.springframework.http.client.SimpleClientHttpRequestFactory;

/**
 * Checks the health of product-service by calling its /actuator/health endpoint.
 *
 * Design rules applied here:
 * - 2s connect / 3s read timeout prevents this check from blocking the health endpoint
 * - try/catch returns DOWN with error detail rather than propagating exceptions
 * - responseTimeMs is included so Spring Boot Admin shows more than just UP/DOWN
 * - Calls /actuator/health, not a business endpoint
 */
@Component
public class ProductServiceHealthIndicator implements HealthIndicator {

    // Read base URL from config so it can be overridden per environment
    @Value("${downstream.product-service.url:http://localhost:8082}")
    private String productServiceUrl;

    @Override
    public Health health() {
        long startTime = System.currentTimeMillis();
        try {
            SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
            factory.setConnectTimeout(2000);   // 2 second connect timeout
            factory.setReadTimeout(3000);      // 3 second read timeout

            RestTemplate restTemplate = new RestTemplate(factory);
            restTemplate.getForObject(productServiceUrl + "/actuator/health", String.class);

            long responseTime = System.currentTimeMillis() - startTime;
            return Health.up()
                .withDetail("url", productServiceUrl)
                .withDetail("responseTimeMs", responseTime)
                .build();

        } catch (RestClientException e) {
            long responseTime = System.currentTimeMillis() - startTime;
            return Health.down()
                .withDetail("url", productServiceUrl)
                .withDetail("error", e.getMessage())
                .withDetail("responseTimeMs", responseTime)
                .build();
        }
    }
}
```

Add the corresponding URL configuration to `application.yml` for each downstream indicator:

```yaml
# Add to application.yml — one entry per downstream service this service calls
downstream:
  product-service:
    url: http://localhost:8082    # Use the actual port found in Step 1 for product-service
  user-service:
    url: http://localhost:8083    # Use the actual port found in Step 1 for user-service
```

**IMPORTANT:** Use the actual `server.port` values you discovered in Step 1 for each service. Do not invent port numbers — read them from each service's `application.yml`.

**Reuse existing config keys — do not duplicate.** Before adding a `downstream.*` key, check whether the service already configures that URL under a different property (e.g., `services.user-service.url`, `telemetry.service.url`). If an existing key already holds the correct URL, reference it directly in the `@Value` annotation:

```java
// ✅ Reuses existing config key — no duplication
@Value("${services.user-service.url:http://localhost:8081}")
private String userServiceUrl;

// ❌ Adds a redundant key when services.user-service.url already exists
@Value("${downstream.user-service.url:http://localhost:8081}")
private String userServiceUrl;
```

Only add a new `downstream.*` key if no suitable URL property already exists in `application.yml`.

### 4c — Database health indicator (if applicable)

If a service has a database and Spring Boot's auto-configured `DataSourceHealthIndicator` is active, no code is needed. Verify it is active by confirming `spring-boot-starter-data-jpa` or `spring-boot-starter-jdbc` is on the classpath.

If a service uses a non-standard database connection not covered by Spring Boot's auto-configuration, write a custom indicator extending `AbstractHealthIndicator`:

```java
@Component
public class DatabaseHealthIndicator extends AbstractHealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    public DatabaseHealthIndicator(JdbcTemplate jdbcTemplate) {
        super("Database health check failed");  // message used when doHealthCheck throws
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        // Validate query runs and returns a result — if it throws, AbstractHealthIndicator catches it
        Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
        builder.up()
            .withDetail("validationQuery", "SELECT 1")
            .withDetail("result", result);
    }
}
```

### 4d — Health groups (separate liveness from readiness)

Add health groups to each service's `application.yml` to clearly separate concerns:

```yaml
management:
  endpoint:
    health:
      group:
        # liveness: only checks the JVM/process — not the database or downstream services
        # A liveness failure causes a Kubernetes restart — don't include DB here
        liveness:
          include: livenessState,diskSpace
          show-details: always
        # readiness: checks everything needed to serve real traffic
        # A readiness failure stops traffic routing but does NOT restart the pod
        readiness:
          include: readinessState,db,userService,productService   # replace with actual indicator IDs
          show-details: always
```

**Use real indicator IDs — not placeholder names.** The `include` list must contain the actual Spring Boot health indicator IDs for each service. The ID is derived from the class name by removing the `HealthIndicator` suffix and lowercasing the first letter:

| Class name | Health indicator ID |
|---|---|
| `UserServiceHealthIndicator` | `userService` |
| `ProductServiceHealthIndicator` | `productService` |
| `TelemetryServiceHealthIndicator` | `telemetryService` |
| `NotificationServiceHealthIndicator` | `notificationService` |

Using a name that does not match any registered indicator (e.g., `downstream`) silently excludes those checks — the readiness group will say UP even when all downstream services are down.

This results in:
- `GET /actuator/health/liveness` → used for Kubernetes liveness probes — never triggers restart due to DB outage
- `GET /actuator/health/readiness` → used for Kubernetes readiness probes — stops traffic if DB or downstream is down
- `GET /actuator/health` → full composite view, consumed by Spring Boot Admin

### 4f — Spring Security and actuator access

If Step 1 discovery found a `SecurityFilterChain` bean in the service, the actuator endpoints are blocked by default for unauthenticated requests. This means:
- `GET /actuator/health` from another service's health indicator will receive **403**, not 200
- The calling service's health indicator will catch the `HttpClientErrorException` and report `DOWN`
- This is **not** the same as the service being unhealthy — it is a misconfiguration

**The fix:** Add `/actuator/**` to the `permitAll()` matcher in the existing `SecurityFilterChain` bean. Do not create a new security config — modify the one that already exists:

```java
http
    .authorizeHttpRequests(authz -> authz
        .requestMatchers("/actuator/**").permitAll()   // ADD THIS LINE
        .requestMatchers("/api/users/register", "/api/users/login").permitAll()
        .anyRequest().authenticated()
    )
```

**How to distinguish 403 from other failures** when a health indicator reports DOWN:

| Error in health details | Root cause | Fix |
|---|---|---|
| `403 : [no body]` | Actuator present but blocked by Spring Security | Add `/actuator/**` to `permitAll()` |
| `404 : {"path":"/actuator/health"}` | Actuator not installed | Add `spring-boot-starter-actuator` dependency |
| `Connection refused` | Target service is not running | Start the service |

This distinction matters: 403 and 404 require code changes; connection refused just means the service needs to be started.

### 4e — GOOD vs BAD health indicator patterns

**GOOD — safe, informative, non-blocking:**
```java
// ✅ Short timeout prevents health endpoint from hanging
// ✅ try/catch returns DOWN instead of propagating exception
// ✅ Detailed error information for diagnostics
// ✅ responseTimeMs makes the check useful for SLA monitoring
@Override
public Health health() {
    long start = System.currentTimeMillis();
    try {
        restTemplate.getForObject(url + "/actuator/health", String.class);
        return Health.up().withDetail("responseTimeMs", System.currentTimeMillis() - start).build();
    } catch (Exception e) {
        return Health.down().withDetail("error", e.getMessage()).build();
    }
}
```

**BAD — dangerous patterns that will break production:**
```java
// ❌ No timeout — hangs if downstream is slow, blocks the health endpoint for 60+ seconds
new RestTemplate().getForObject(url, String.class);

// ❌ Throws exception instead of returning Health.down() — causes 500 on /actuator/health
// ❌ Breaks Spring Boot Admin's ability to read health status
throw new RuntimeException("service unavailable");

// ❌ Calling a business endpoint instead of /actuator/health
// Business endpoints have different auth, rate limiting, and side effects
restTemplate.getForObject(url + "/api/products", String.class);

// ❌ No error detail — DOWN with no context is useless for debugging
return Health.down().build();
```

---

## STEP 5 — CREATE admin-service FROM SCRATCH

Create a new directory `admin-service/` at the working directory root. This is a brand-new Spring Boot application.

### Directory structure to create:

```
admin-service/
├── build.gradle
├── gradlew                    (copy from any existing service)
├── gradlew.bat                (copy from any existing service)
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar    (copy from any existing service)
│       └── gradle-wrapper.properties (copy from any existing service)
└── src/
    └── main/
        ├── java/
        │   └── com/example/adminservice/
        │       └── AdminServiceApplication.java
        └── resources/
            └── application.yml
```

After creating the directory and copying the Gradle wrapper files, initialise git and create the working branch:

```bash
cd admin-service
git init
git checkout -b ai-single-prompt-health
cd ..
```

All subsequent files written to `admin-service` are committed on `ai-single-prompt-health`, not `main`.

### `build.gradle` for admin-service:

```groovy
plugins {
    id 'org.springframework.boot' version '<SAME VERSION AS OTHER SERVICES — READ FROM build.gradle>'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

ext {
    set('springBootAdminVersion', '3.3.5')
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'de.codecentric:spring-boot-admin-starter-server'

    // Spring Boot Admin security — show the login page, not open access
    implementation 'org.springframework.boot:spring-boot-starter-security'
}

dependencyManagement {
    imports {
        mavenBom "de.codecentric:spring-boot-admin-dependencies:${springBootAdminVersion}"
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### `AdminServiceApplication.java`:

```java
package com.example.adminservice;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;

@SpringBootApplication
@EnableAdminServer
public class AdminServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServiceApplication.class, args);
    }

    /**
     * Security configuration: protect the admin UI with basic auth.
     * The six client services register using these credentials.
     * For this demonstration environment: admin / admin.
     */
    @Configuration
    static class SecurityConfig {

        @Bean
        SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            SavedRequestAwareAuthenticationSuccessHandler successHandler =
                new SavedRequestAwareAuthenticationSuccessHandler();
            successHandler.setTargetUrlParameter("redirectTo");
            successHandler.setDefaultTargetUrl("/");

            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/assets/**", "/login").permitAll()
                    .anyRequest().authenticated()
                )
                .formLogin(form -> form
                    .loginPage("/login")
                    .successHandler(successHandler)
                )
                .logout(logout -> logout.logoutUrl("/logout"))
                .httpBasic(org.springframework.security.config.Customizer.withDefaults())
                .csrf(csrf -> csrf
                    .csrfTokenRepository(
                        org.springframework.security.web.csrf.CookieCsrfTokenRepository.withHttpOnlyFalse()
                    )
                    .ignoringRequestMatchers(
                        // Allow the SBA client registration endpoint to work without CSRF token
                        new org.springframework.security.web.util.matcher.AntPathRequestMatcher(
                            "/instances", "POST"
                        ),
                        new org.springframework.security.web.util.matcher.AntPathRequestMatcher(
                            "/instances/*", "DELETE"
                        ),
                        new org.springframework.security.web.util.matcher.AntPathRequestMatcher(
                            "/actuator/**"
                        )
                    )
                );
            return http.build();
        }
    }
}
```

### `application.yml` for admin-service:

```yaml
server:
  port: 8099    # admin-service port — must match spring.boot.admin.client.url in all six services

spring:
  application:
    name: admin-service
  security:
    user:
      name: admin       # credentials for the Spring Boot Admin UI
      password: admin   # for this demonstration only — use secrets management in production

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always

info:
  app:
    name: admin-service
    description: Spring Boot Admin dashboard — monitors all six microservices
    version: 1.0.0

logging:
  level:
    de.codecentric.boot.admin: INFO
```

### Update client registration configuration in all six services

Because `admin-service` is now secured with HTTP Basic auth, each client's `application.yml` registration block must include the credentials:

```yaml
# Add to each service's application.yml — update the spring.boot.admin.client section
spring:
  boot:
    admin:
      client:
        url: http://localhost:8099
        username: admin      # must match spring.security.user.name in admin-service
        password: admin      # must match spring.security.user.password in admin-service
        instance:
          name: ${spring.application.name}
          prefer-ip: true
          metadata:
            tags:
              environment: local
```

---

## STEP 6 — HEALTH INDICATOR TESTS

For each custom health indicator created in Step 4, write a unit test at:

`src/test/java/{base_package}/health/{IndicatorName}Test.java`

```java
package com.example.orderservice.health;

import org.junit.jupiter.api.Test;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.Status;
import org.springframework.test.util.ReflectionTestUtils;
import org.mockito.Mockito;
import org.springframework.web.client.ResourceAccessException;

import static org.assertj.core.api.Assertions.assertThat;

class ProductServiceHealthIndicatorTest {

    @Test
    void shouldReturnUpWhenProductServiceIsReachable() {
        // Use WireMock or a simple mock — stub the RestTemplate call
        ProductServiceHealthIndicator indicator = new ProductServiceHealthIndicator();
        ReflectionTestUtils.setField(indicator, "productServiceUrl", "http://localhost:9999");

        // For a pure unit test, override the RestTemplate to return 200
        // In an integration test, use @SpringBootTest + MockRestServiceServer
        Health health = indicator.health(); // Will return DOWN in unit test environment — that's expected
        assertThat(health).isNotNull();
        assertThat(health.getDetails()).containsKey("url");
        assertThat(health.getDetails()).containsKey("responseTimeMs");
    }

    @Test
    void shouldReturnDownWhenProductServiceIsUnreachable() {
        ProductServiceHealthIndicator indicator = new ProductServiceHealthIndicator();
        ReflectionTestUtils.setField(indicator, "productServiceUrl", "http://localhost:1");  // port 1 always refused
        Health health = indicator.health();
        assertThat(health.getStatus()).isEqualTo(Status.DOWN);
        assertThat(health.getDetails()).containsKey("error");
        assertThat(health.getDetails()).containsKey("responseTimeMs");
    }
}
```

---

## STEP 7 — GITHUB ACTIONS WORKFLOW

For each of the six existing services, add or update `.github/workflows/health.yml`:

```yaml
name: Health Endpoint Check

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build and test
        run: ./gradlew build
      - name: Verify actuator health config is present
        run: |
          grep -r "actuator" src/main/resources/ || \
            (echo "ERROR: management.endpoints.web.exposure missing from application.yml" && exit 1)
          grep -r "show-details: always" src/main/resources/ || \
            (echo "ERROR: show-details: always missing from application.yml" && exit 1)
          grep -r "admin-service" src/main/resources/ || \
            (echo "ERROR: Spring Boot Admin client URL missing from application.yml" && exit 1)
```

For `admin-service`, create `.github/workflows/build.yml`:

```yaml
name: Admin Service Build

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build admin-service
        run: |
          cd admin-service
          ./gradlew build
```

---

## STEP 8 — VERIFICATION

After all changes are applied, verify by running:

```bash
# Verify each service builds
for svc in notification-service order-service payment-service product-service telemetry-service user-service admin-service; do
    echo "=== Building $svc ==="
    cd $svc && ./gradlew build && cd ..
done

# Verify health config is consistent across all six services
echo "=== Checking show-details consistency ==="
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: show-details = "
    grep "show-details" $svc/src/main/resources/application.yml
done

echo "=== Checking git info config consistency ==="
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: git.enabled = "
    grep "enabled: true" $svc/src/main/resources/application.yml | grep -A1 "git:" || echo "MISSING"
    echo -n "$svc: git.mode = "
    grep "mode: full" $svc/src/main/resources/application.yml || echo "MISSING"
done

echo "=== Checking git.properties is generated after build ==="
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: "
    ls $svc/build/resources/main/git.properties 2>/dev/null && echo "OK" || echo "MISSING — run ./gradlew build first"
done

# After starting all services, verify git info appears at the info endpoint
# curl -s http://localhost:<port>/actuator/info | python3 -m json.tool
# Expected: a "git" object containing branch, commit.id, commit.time, etc.

# After starting all services, verify the admin dashboard
# 1. Start all six services
# 2. Start admin-service: cd admin-service && ./gradlew bootRun
# 3. Navigate to http://localhost:8099 — login with admin/admin
# 4. Confirm all six services appear as registered and ONLINE
# 5. Click each service — confirm health details show db, downstream, diskSpace components
# 6. Hit each service's health endpoint directly:
curl -s http://localhost:<port>/actuator/health | python3 -m json.tool

# Diagnosing DOWN components — check the "error" field in the health response:
#
#   "error": "403 : [no body]"
#     → Actuator is present but Spring Security is blocking unauthenticated access.
#       Fix: add .requestMatchers("/actuator/**").permitAll() to the SecurityFilterChain.
#
#   "error": "404 : {\"path\":\"/actuator/health\"}"
#     → Actuator is not installed on the target service.
#       Fix: add spring-boot-starter-actuator to that service's build.gradle.
#
#   "error": "... Connection refused"
#     → The target service is not running.
#       Fix: start the service.
#
# A service showing DOWN because one of its dependencies is not running is expected
# and correct behaviour — it does not indicate a bug in the health indicator.
```

### 8b — Automated status and component validation

After all seven services are started, run the following script to confirm every service reports overall status `UP` and exposes the expected health check components.

**Before running:** replace each `<PORT>` with the `server.port` value you recorded in Step 1.

```bash
#!/usr/bin/env bash
# validate-health.sh — run from the working directory root after all services are started

# ── Port map: fill in the actual ports discovered in Step 1 ──────────────────
declare -A PORT_MAP=(
  [notification-service]=<PORT>
  [order-service]=<PORT>
  [payment-service]=<PORT>
  [product-service]=<PORT>
  [telemetry-service]=<PORT>
  [user-service]=<PORT>
)

# ── Expected component map: fill in based on Step 1 discovery ────────────────
# Format: "comp1 comp2 comp3"
# Every service gets diskSpace + ping + livenessState + readinessState as a baseline.
# Add "db" for services with a DataSource.
# Add the camelCase indicator ID for every custom HealthIndicator (e.g. "userService").
declare -A EXPECTED_COMPONENTS=(
  [notification-service]="diskSpace ping livenessState readinessState"
  [order-service]="diskSpace ping livenessState readinessState"
  [payment-service]="diskSpace ping livenessState readinessState"
  [product-service]="diskSpace ping livenessState readinessState"
  [telemetry-service]="diskSpace ping livenessState readinessState"
  [user-service]="diskSpace ping livenessState readinessState"
)
# Example after filling in for a service that has a DB and calls two downstream services:
#   [order-service]="diskSpace ping livenessState readinessState db productService userService"

FAILED=0

for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
  PORT=${PORT_MAP[$svc]}
  echo "──────────────────────────────────────────"
  echo "Checking $svc on port $PORT"

  RESPONSE=$(curl -s --max-time 5 "http://localhost:$PORT/actuator/health" 2>/dev/null)

  if [ -z "$RESPONSE" ]; then
    echo "  FAIL — no response (is the service running?)"
    FAILED=1
    continue
  fi

  # Overall status must be UP
  OVERALL=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','MISSING'))" 2>/dev/null)
  if [ "$OVERALL" != "UP" ]; then
    echo "  FAIL — overall status is '$OVERALL' (expected UP)"
    echo "$RESPONSE" | python3 -m json.tool 2>/dev/null
    FAILED=1
  else
    echo "  OK   — overall status: UP"
  fi

  # Retrieve component names and statuses
  COMP_JSON=$(echo "$RESPONSE" | python3 -c "
import sys, json
d = json.load(sys.stdin)
comps = d.get('components', {})
for name, val in sorted(comps.items()):
    print(name, val.get('status', 'UNKNOWN'))
" 2>/dev/null)

  COMP_COUNT=$(echo "$COMP_JSON" | wc -l | tr -d ' ')
  echo "  Components found ($COMP_COUNT):"
  while IFS= read -r line; do
    COMP_NAME=$(echo "$line" | awk '{print $1}')
    COMP_STATUS=$(echo "$line" | awk '{print $2}')
    if [ "$COMP_STATUS" = "UP" ]; then
      echo "    ✓ $COMP_NAME: $COMP_STATUS"
    else
      echo "    ✗ $COMP_NAME: $COMP_STATUS  ← DEGRADED"
      FAILED=1
    fi
  done <<< "$COMP_JSON"

  # Verify every expected component is present
  for comp in ${EXPECTED_COMPONENTS[$svc]}; do
    FOUND=$(echo "$COMP_JSON" | awk '{print $1}' | grep -c "^${comp}$" || true)
    if [ "$FOUND" -eq 0 ]; then
      echo "  FAIL — expected component '$comp' is missing from /actuator/health response"
      FAILED=1
    fi
  done
done

echo "══════════════════════════════════════════"
if [ $FAILED -eq 0 ]; then
  echo "PASSED — all services UP with expected components."
else
  echo "FAILED — see errors above. Common fixes:"
  echo "  • 'MISSING' component  → HealthIndicator class not annotated @Component, or indicator ID in health group is wrong"
  echo "  • Component status DOWN → check 'error' field in the component details for 403 / 404 / connection-refused"
  echo "  • No response          → service is not running"
  exit 1
fi
```

**What this script checks:**

| Check | Pass condition |
|---|---|
| Overall status | `"status": "UP"` for every service |
| Component presence | Every component listed in `EXPECTED_COMPONENTS` exists in the response |
| Component status | Every component reports `UP` (degraded components are flagged) |
| Component count | Printed for visual inspection — use to catch unexpectedly missing indicators |

**Fill in `EXPECTED_COMPONENTS` correctly.** Use the Health Readiness Map from Step 1 to determine which entries belong in each service:

| Indicator | When to include |
|---|---|
| `db` | Service has a `DataSource` (JPA, JDBC, H2, PostgreSQL, etc.) |
| `diskSpace` | Always — built in |
| `ping` | Always — built in |
| `livenessState` | Always — enabled in Step 3 |
| `readinessState` | Always — enabled in Step 3 |
| `userService` | Service has a `UserServiceHealthIndicator` (class name → strip `HealthIndicator`, lowercase first letter) |
| `productService` | Service has a `ProductServiceHealthIndicator` |
| `notificationService` | Service has a `NotificationServiceHealthIndicator` |
| *(etc.)* | One entry per custom `HealthIndicator` class |

---

## DEFINITION OF DONE

**Across all six existing services:**
- [ ] `spring-boot-starter-actuator` is present in every `build.gradle`
- [ ] `de.codecentric:spring-boot-admin-starter-client:3.3.5` is present in every `build.gradle`
- [ ] `com.gorylenko.gradle-git-properties` plugin version `2.4.1` is present in every `build.gradle`
- [ ] `gitProperties {}` block is present in every `build.gradle` with the full key list
- [ ] `management.endpoint.health.show-details: always` is in every `application.yml`
- [ ] `management.endpoints.web.exposure.include: health,info,metrics,loggers` is identical in every `application.yml`
- [ ] `spring.boot.admin.client.url: http://localhost:8099` is identical in every `application.yml` (no trailing slash)
- [ ] `management.info.git.enabled: true` is in every `application.yml`
- [ ] `management.info.git.mode: full` is in every `application.yml`
- [ ] `./gradlew build` generates `build/resources/main/git.properties` in every service
- [ ] `GET /actuator/info` returns a `git` object with `branch`, `commit.id`, and `commit.time` fields
- [ ] The `git.branch` value visible in Spring Boot Admin matches the branch each service was built from
- [ ] Every service that has a `DataSource` exposes a DB health indicator (built-in or custom)
- [ ] Every service that calls another service has a custom downstream `HealthIndicator` with 2s/3s timeout, try/catch, and `responseTimeMs` detail
- [ ] Health groups separate `liveness` (diskSpace only) from `readiness` (db + downstream)
- [ ] `./gradlew build` passes for all six services
- [ ] Unit tests exist for every custom `HealthIndicator`
- [ ] `GET /actuator/health` returns `{"status":"UP"}` for every service when all dependencies are running
- [ ] Every service exposes at minimum: `diskSpace`, `ping`, `livenessState`, `readinessState` components
- [ ] Services with a `DataSource` include a `db` component in the health response
- [ ] Services with custom `HealthIndicator` classes include the corresponding camelCase component ID in the health response
- [ ] The `validate-health.sh` script from Step 8b exits 0 (all services UP, all expected components present)

**For admin-service:**
- [ ] `admin-service/` directory exists in the working directory root
- [ ] `build.gradle` uses the same Spring Boot version as the other services
- [ ] `AdminServiceApplication.java` has `@EnableAdminServer`
- [ ] Admin UI is accessible at `http://localhost:8099` with login `admin` / `admin`
- [ ] All six services appear as registered and ONLINE in the admin dashboard when running locally
- [ ] Health details (db, downstream, diskSpace) are visible per service in the dashboard

**GitHub Actions:**
- [ ] A health workflow exists in every repository (including `admin-service`)

---

## CRITICAL CONSTRAINTS

1. **Do not change `spring.application.name`** in any of the six services — this is the identity shown in the admin dashboard.
2. **`spring.boot.admin.client.url` must be `http://localhost:8099` in all six services** — identical string, no trailing slash, no variation.
3. **`management.endpoint.health.show-details` must be `always` in all six services** — Spring Boot Admin cannot read health components with any other value.
4. **Downstream health indicators must have a timeout** — no exceptions. An indicator without a timeout can cause `/actuator/health` to hang for 60+ seconds, which breaks the admin dashboard's polling and makes every service appear offline.
5. **Never throw an uncaught exception from a `HealthIndicator`** — always return `Health.down().withDetail("error", message).build()` instead.
6. **The `admin-service` port must be `8099`** — this must match the client URL configured in all six services. Verify this is not already taken by any of the six services before finalising.
7. **Do not use `management.endpoints.web.exposure.include: "*"`** in the six services — wildcard exposure is a security risk. Expose only `health,info,metrics,loggers`.
8. **All changes go on `ai-single-prompt-health`** — this branch is created in Step 0 for the six existing repos, and after `git init` for `admin-service` in Step 5. Never commit to `main` in any repo. Verify the active branch before making any commit with `git branch --show-current`.
9. **Check the actual `server.port` of each service from its `application.yml`** before writing downstream health indicator URLs — do not invent port numbers or assume defaults.
10. **If a service has Spring Security, `/actuator/**` must be explicitly permitted** — add `.requestMatchers("/actuator/**").permitAll()` to the existing `SecurityFilterChain` bean. Without this, all downstream health indicators that call this service will receive 403 and report DOWN, even when the service is fully healthy. See Step 4f.
11. **The `git-commit-id` plugin requires a `.git` directory** — each repo must be a git repository (which they are, since Step 0 checks out a branch). The plugin will fail with `dotGitDirectory does not exist` if run outside a git repo. Do not run `./gradlew build` from a path that does not have a `.git` ancestor.