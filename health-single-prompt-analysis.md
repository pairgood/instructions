# Health Endpoint Implementation Analysis: `ai-single-prompt-health` vs `manual-health`

**Reference implementation:** `manual-health`
**Implementation under review:** `ai-single-prompt-health`
**Services analyzed:** notification-service, order-service, payment-service, product-service, telemetry-service, user-service

Only differences that result in an inaccurate or incorrect implementation are reported. Cosmetic differences (dependency ordering in `build.gradle`, added Javadoc comments, `info.app.description` string values, `requestMatchers` rule reordering where all rules use `permitAll()`) are explicitly excluded.

---

## Per-Service Analysis

### 1. notification-service

#### `.github/workflows/health.yml` — INCORRECT

The Java distribution is set to `temurin` instead of the required `corretto`.

```yaml
# manual-health (correct)
distribution: 'corretto'

# ai-single-prompt-health (incorrect)
distribution: 'temurin'
```

---

#### `src/main/resources/application.yml` — INCORRECT

The AI branch is missing the `downstream.telemetry-service.url` configuration property entirely. The correct implementation defines:

```yaml
# manual-health (correct)
downstream:
  telemetry-service:
    url: http://localhost:8086
```

The AI branch omits the entire `downstream` block, so the telemetry service URL cannot be overridden through environment-specific configuration.

---

#### `src/main/java/.../health/TelemetryServiceHealthIndicator.java` — INCORRECT

The `@Value` annotation references the wrong configuration property key:

```java
// manual-health (correct) — matches the key defined in application.yml
@Value("${downstream.telemetry-service.url:http://localhost:8086}")
private String telemetryServiceUrl;

// ai-single-prompt-health (incorrect) — references a key not defined in application.yml
@Value("${telemetry.service.url:http://localhost:8086}")
private String telemetryServiceUrl;
```

Combined with the missing `downstream` block in `application.yml`, the indicator always falls back to its hardcoded default URL and cannot be configured per environment. This would silently produce incorrect health check behavior in staging or production.

---

#### `src/test/java/.../health/TelemetryServiceHealthIndicatorTest.java` — INCORRECT

The AI test suite injects the URL directly via `ReflectionTestUtils.setField`, bypassing the `@Value` annotation entirely. This means the config key mismatch described above is never detected by tests — the implementation compiles, tests pass, but the indicator silently uses the wrong property key in a deployed service.

Additionally, the manual `shouldReturnDetailsWhenHealthCheckRuns` test (which verifies that `url` and `responseTimeMs` details are always populated regardless of outcome) is absent. All AI tests exclusively exercise the DOWN path through port 1, providing no coverage of the `Health.up()` detail population path.

---

### 2. order-service

#### `.github/workflows/health.yml` — INCORRECT

Same as all other services: `distribution: 'temurin'` instead of `distribution: 'corretto'`.

---

#### `src/test/java/.../health/*HealthIndicatorTest.java` — INCORRECT

Applies to all four indicator tests (`NotificationServiceHealthIndicatorTest`, `ProductServiceHealthIndicatorTest`, `TelemetryServiceHealthIndicatorTest`, `UserServiceHealthIndicatorTest`).

The manual implementation includes `shouldReturnDetailsWhenHealthCheckRuns` which uses port 9999, asserts detail keys (`url`, `responseTimeMs`) are present, and does not assert a specific `Status`. This verifies that details are always populated regardless of outcome.

The AI version replaces this with two tests that both use port 1 (unreachable), meaning the `Health.up()` detail population path is never exercised by any test across all four indicators.

---

### 3. payment-service

#### `.github/workflows/health.yml` — INCORRECT

Same as all other services: `distribution: 'temurin'` instead of `distribution: 'corretto'`.

---

#### `src/test/java/.../health/*HealthIndicatorTest.java` — INCORRECT

Same structural gap as order-service, affecting both `NotificationServiceHealthIndicatorTest` and `TelemetryServiceHealthIndicatorTest`. The `shouldReturnDetailsWhenHealthCheckRuns` test is absent; all AI tests exercise only the DOWN path.

---

### 4. product-service

#### `.github/workflows/health.yml` — INCORRECT

Same as all other services: `distribution: 'temurin'` instead of `distribution: 'corretto'`.

---

#### `src/test/java/.../health/TelemetryServiceHealthIndicatorTest.java` — INCORRECT

Same structural gap: `shouldReturnDetailsWhenHealthCheckRuns` is absent. Both AI tests use port 1 and exercise only the DOWN path.

---

### 5. telemetry-service

#### `.github/workflows/health.yml` — INCORRECT

Same as all other services: `distribution: 'temurin'` instead of `distribution: 'corretto'`.

telemetry-service has no `HealthIndicator` classes or tests (correct, as it has no downstream dependencies). The only functional difference is the CI workflow JDK distribution.

---

### 6. user-service

#### `.github/workflows/health.yml` — INCORRECT

Same as all other services: `distribution: 'temurin'` instead of `distribution: 'corretto'`.

---

#### `src/test/java/.../health/TelemetryServiceHealthIndicatorTest.java` — INCORRECT

Same structural gap: `shouldReturnDetailsWhenHealthCheckRuns` is absent. Both AI tests use port 1 and exercise only the DOWN path.

---

## Summary of Error Patterns

### Pattern 1: Wrong JDK distribution in all 6 services

**Affected:** All 6 services — `.github/workflows/health.yml`

The AI implementation consistently uses `distribution: 'temurin'` in the `actions/setup-java@v4` step. Every service in the correct implementation uses `distribution: 'corretto'`. This affects every CI build across all services.

---

### Pattern 2: Wrong `@Value` config key (notification-service only)

**Affected:** notification-service — `TelemetryServiceHealthIndicator.java` and `application.yml`

notification-service uniquely uses a `downstream.telemetry-service.url` config key. The AI used `telemetry.service.url` (the pattern from other services) and also omitted the `downstream` block from `application.yml`. The indicator silently falls back to its hardcoded default and cannot be configured per environment. Tests pass due to direct field injection, masking the bug.

---

### Pattern 3: Missing `shouldReturnDetailsWhenHealthCheckRuns` test in 5 of 6 services

**Affected:** notification-service, order-service (4 indicators), payment-service (2 indicators), product-service, user-service

The manual implementation consistently includes a test that verifies the `Health.up()` detail population path by connecting to port 9999 and asserting that `url` and `responseTimeMs` keys are always present in the result. The AI version uses only port 1 for all tests, exclusively exercising the error/DOWN path. The UP path is never covered by any test.

---

## Consolidated Error Table

| Service | File | Error | Description |
|---|---|---|---|
| All 6 | `.github/workflows/health.yml` | Wrong JDK distribution | `distribution: 'temurin'` instead of `distribution: 'corretto'` |
| notification-service | `TelemetryServiceHealthIndicator.java` | Wrong `@Value` config key | Uses `${telemetry.service.url:...}` instead of `${downstream.telemetry-service.url:...}` |
| notification-service | `application.yml` | Missing configuration property | `downstream.telemetry-service.url` block is absent |
| notification-service | `TelemetryServiceHealthIndicatorTest.java` | Missing test coverage + config key bug masked | No UP-path test; field injection hides the `@Value` key mismatch |
| order-service | All 4 `*HealthIndicatorTest.java` | Missing test coverage | No test exercises `Health.up()` detail population for any of the 4 indicators |
| payment-service | Both `*HealthIndicatorTest.java` | Missing test coverage | No test exercises `Health.up()` detail population for either indicator |
| product-service | `TelemetryServiceHealthIndicatorTest.java` | Missing test coverage | No test exercises `Health.up()` detail population |
| user-service | `TelemetryServiceHealthIndicatorTest.java` | Missing test coverage | No test exercises `Health.up()` detail population |
