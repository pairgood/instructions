# NAIVE PROMPT — DISTRIBUTED TRACING (Jaeger + OpenTelemetry)
## For use against: notification-service, order-service, payment-service, product-service, telemetry-service, user-service

---

You are an expert Spring Boot observability engineer. You will implement distributed tracing using OpenTelemetry and Jaeger across all six microservices in this working directory. This is an existing codebase — read before writing anything.

## THE SIX SERVICES

The working directory contains six Spring Boot + Gradle microservices:
- `notification-service`
- `order-service`
- `payment-service`
- `product-service`
- `telemetry-service`
- `user-service`

## INFRASTRUCTURE (ALREADY RUNNING — DO NOT MODIFY)

- **Jaeger all-in-one**: `http://localhost:16686` (UI), OTLP gRPC endpoint: `http://localhost:4317`, OTLP HTTP endpoint: `http://localhost:4318`
- **Java**: 17+, **Spring Boot**: 3.x, **Build tool**: Gradle (Groovy DSL)
- **Tracing approach**: Micrometer Tracing Bridge OTel + OpenTelemetry Java SDK, exporting via OTLP to Jaeger

---

## STEP 1 — DISCOVERY (READ EVERYTHING BEFORE WRITING ANYTHING)

For **each** of the six services, read and record:

1. The full `build.gradle` — note existing dependencies, Spring Boot version (determines which Micrometer tracing BOM to use), and any existing observability-related dependencies
2. Every `src/main/resources/application.yml` and `application-*.yml` — record `spring.application.name`, `server.port`, and any existing `management.*` or `logging.*` configuration
3. Every `@RestController` — list each endpoint path and HTTP method
4. Every class that calls another service via `RestTemplate`, `WebClient`, `FeignClient`, or `RestClient` — these require trace context propagation configuration
5. Any existing `@Async` methods, `ThreadPoolTaskExecutor` beans, or `CompletableFuture` usage — async calls require explicit MDC context propagation
6. Any existing logging configuration files (`logback-spring.xml`, `logback.xml`, `log4j2.xml`)

Write a complete **observability readiness map** before touching any file:

```
OBSERVABILITY READINESS MAP
============================

[service-name]
  spring.application.name: <value>
  server.port: <value>
  Spring Boot version: <value>
  Existing micrometer/tracing deps: <list or NONE FOUND>
  Existing logging config file: <path or NONE FOUND>
  Outbound HTTP clients: <RestTemplate/WebClient/Feign or NONE>
  Async methods: <list or NONE FOUND>
  @RestController endpoints: <list>
```

**Do not proceed to Step 2 until the readiness map is complete for all six services.**

---

## STEP 2 — DEPENDENCY ADDITIONS

Add the following to **every** service's `build.gradle`, inside the existing `dependencies {}` block. Do not alter any existing lines. Do not specify explicit versions for dependencies managed by the Spring Boot BOM — the BOM version is inherited from the `org.springframework.boot` plugin.

```groovy
// OpenTelemetry tracing — add inside existing dependencies block

// Micrometer Tracing bridge for OpenTelemetry (BOM-managed in Spring Boot 3.x)
implementation 'io.micrometer:micrometer-tracing-bridge-otel'

// OpenTelemetry exporter — OTLP gRPC to Jaeger
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'

// Required for OTLP gRPC transport
implementation 'io.grpc:grpc-netty-shaded:1.63.0'

// Actuator exposes tracing metrics endpoints
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

If the service uses `RestTemplate` for outbound calls, also add:
```groovy
// Context propagation for RestTemplate
implementation 'io.micrometer:micrometer-tracing'
```

If the service uses `WebClient`, the Spring WebFlux starter includes propagation automatically — no additional dependency needed.

---

## STEP 3 — CONFIGURATION ADDITIONS

Add the following to **every** service's `application.yml`. Merge carefully — do not overwrite existing keys. Use YAML block style, not inline JSON.

```yaml
# OpenTelemetry / Jaeger tracing — merge into existing application.yml
management:
  tracing:
    sampling:
      probability: 1.0   # 100% sampling for demonstration — reduce to 0.1 in production
  otlp:
    tracing:
      endpoint: http://127.0.0.1:4318/v1/traces  # Jaeger OTLP HTTP endpoint — SAME value in every service

spring:
  application:
    name: <READ FROM EXISTING yml — do not change>  # MUST already exist; do not add if present

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n"
```

**Consistency rules — enforce across all six services:**
- `management.tracing.sampling.probability` must be `1.0` in all six services
- `management.otlp.tracing.endpoint` must be `http://127.0.0.1:4318/v1/traces` in all six services (identical string)
- `spring.application.name` values must NOT be modified — use whatever is already there
- The log pattern's `[%X{traceId:-},%X{spanId:-}]` format must be identical in all six services

> **Why `127.0.0.1` and port `4318`?**
> Spring Boot's `management.otlp.tracing.endpoint` uses HTTP/protobuf transport (OkHttp), not gRPC.
> Port 4317 is gRPC-only — sending HTTP to it causes "Broken pipe" errors and silent span loss.
> Port 4318 is the OTLP HTTP port; the full path `/v1/traces` must be included.
> Use `127.0.0.1` rather than `localhost` — on macOS, `localhost` resolves to IPv6 (`::1`) but
> Jaeger binds IPv4 only, causing "Connection refused" errors.

---

## STEP 4 — TRACE CONTEXT PROPAGATION CONFIGURATION

Create a `TracingConfig.java` (or `ObservabilityConfig.java`) configuration class in the main package of **each** service that makes outbound HTTP calls.

### For services using RestTemplate:

```java
package com.example.orderservice.config;

import io.micrometer.observation.ObservationRegistry;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class TracingConfig {

    /**
     * RestTemplate bean with ObservationRegistry wired in.
     * This enables automatic W3C TraceContext header propagation (traceparent/tracestate)
     * on every outbound HTTP call made through this RestTemplate.
     *
     * IMPORTANT: Every service that calls another service MUST use this bean,
     * not a manually constructed new RestTemplate(). Constructing RestTemplate
     * manually bypasses trace propagation entirely.
     */
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder, ObservationRegistry registry) {
        return builder
            .observationRegistry(registry)
            .build();
    }
}
```

### For services using WebClient:

```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    // WebClient.Builder in Spring Boot 3.x auto-configures OTel propagation
    // when the micrometer-tracing-bridge-otel dependency is on the classpath.
    // Just inject the builder — do not construct WebClient.create() directly.
    return builder.build();
}
```

### For services with @Async methods (if any found in Step 1):

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        // Wrap with ContextPropagatingTaskDecorator to carry MDC (traceId/spanId) into async threads
        executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
        executor.initialize();
        return executor;
    }
}
```

---

## STEP 5 — LOG PATTERN CONFIGURATION

If a service already has a `logback-spring.xml` or `logback.xml`, **do not overwrite it**. Instead, add the `traceId` and `spanId` MDC keys to the existing appender pattern using the same `%X{traceId:-}` MDC syntax.

If no custom Logback config file exists, the `logging.pattern.console` property in `application.yml` (set in Step 3) is sufficient — do not create a `logback-spring.xml` in that case.

If the service has a `logback-spring.xml`, modify the existing `<pattern>` or `<Pattern>` element to include `[%X{traceId:-},%X{spanId:-}]` immediately before the logger name:

```xml
<!-- Add traceId/spanId to existing pattern — example only, adapt to actual pattern found -->
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n</pattern>
```

---

## STEP 6 — GITHUB ACTIONS OBSERVABILITY WORKFLOW

For each service, add or update `.github/workflows/observability.yml`:

```yaml
name: Observability Build Check

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
      - name: Verify tracing configuration is present
        run: |
          # Fail the build if OTLP endpoint configuration is missing from application.yml
          grep -r "otlp" src/main/resources/ || (echo "ERROR: OTel OTLP configuration missing" && exit 1)
          grep -r "traceId" src/main/resources/ || (echo "ERROR: traceId log pattern missing" && exit 1)
```

---

## STEP 7 — VERIFICATION

After applying all changes to all six services, verify the implementation by running each service locally and confirming:

```bash
# Build check — run in each service directory
cd <service-name>
./gradlew build

# Confirm tracing config is present in each application.yml
grep -A3 "tracing:" src/main/resources/application.yml
grep "otlp" src/main/resources/application.yml
grep "traceId" src/main/resources/application.yml

# After starting all services, send a test request that crosses service boundaries
# Then check Jaeger UI at http://localhost:16686
# Search for service: <any service name from spring.application.name>
# Verify: the trace shows spans from multiple services connected by the same traceId
```

> **Gradle bootRun and config changes**: `./gradlew bootRun` serves `application.yml` from
> `build/resources/main/`, not `src/main/resources/`. If you edit `src/main/resources/application.yml`
> after starting the service, the running process does NOT see the change. You must stop the service
> and run `./gradlew bootRun` again (which triggers `processResources`) for the new config to take effect.
> Alternatively, copy the updated file to `build/resources/main/application.yml` before restarting.

### Consistency verification — run across all six service directories:

```bash
# All six should return the same endpoint value
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: "
    grep "endpoint" $svc/src/main/resources/application.yml
done

# All six should return 1.0
for svc in notification-service order-service payment-service product-service telemetry-service user-service; do
    echo -n "$svc: "
    grep "probability" $svc/src/main/resources/application.yml
done
```

### Definition of Done

- [ ] All six `application.yml` files contain `management.tracing.sampling.probability: 1.0`
- [ ] All six `application.yml` files contain `management.otlp.tracing.endpoint: http://127.0.0.1:4318/v1/traces` (identical string)
- [ ] All six `application.yml` files have the log pattern including `[%X{traceId:-},%X{spanId:-}]`
- [ ] All six `build.gradle` files contain `micrometer-tracing-bridge-otel` and `opentelemetry-exporter-otlp`
- [ ] Every service that makes outbound HTTP calls has a `@Bean` RestTemplate or WebClient wired with `ObservationRegistry`
- [ ] `./gradlew build` passes for all six services
- [ ] Jaeger UI at `http://localhost:16686` shows traces from all six services after a request flow
- [ ] A single trace in Jaeger shows connected spans from all services that participate in a given call chain
- [ ] `traceId` is visible in log output for each service and matches the Jaeger trace ID
- [ ] GitHub Actions workflows exist in every repository

---

## CRITICAL CONSTRAINTS

1. **Do not change `spring.application.name`** in any service. These names are the service identity in Jaeger.
2. **`management.otlp.tracing.endpoint` must be identical in all six services** — `http://127.0.0.1:4318/v1/traces`, no variation.
3. **`management.tracing.sampling.probability` must be `1.0` in all six services** — same value, no variation.
4. **Do not modify any existing production logic.** Only add configuration, dependencies, and the `TracingConfig.java` class.
5. **Do not add a `logback-spring.xml`** if one does not already exist — use `logging.pattern.console` in `application.yml` instead.
6. **Use `RestTemplateBuilder.observationRegistry()`** for RestTemplate, not manual interceptor wiring. Manual `ClientHttpRequestInterceptor` approaches do not propagate W3C `traceparent` headers correctly with Micrometer Tracing.
7. **One branch for all changes: `feature/ai-naive-tracing`** — create this branch before writing any code, one per repo.
