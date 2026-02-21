# NAIVE PROMPT — PACT CONTRACT TESTING
## For use against: notification-service, order-service, payment-service, product-service, telemetry-service, user-service

---

You are an expert Spring Boot and consumer-driven contract testing engineer. You will implement comprehensive Pact contract tests across all six microservices in this working directory. This is an existing codebase — read before writing anything.

## THE SIX SERVICES

The working directory contains six Spring Boot + Gradle microservices:
- `notification-service`
- `order-service`
- `payment-service`
- `product-service`
- `telemetry-service`
- `user-service`

## INFRASTRUCTURE (ALREADY RUNNING — DO NOT MODIFY)

- **Pact Broker**: `http://localhost:9292` — credentials `admin` / `admin`
- **Java**: 17+, **Spring Boot**: 3.x, **Build tool**: Gradle (Groovy DSL)
- **Pact JVM version**: `au.com.dius.pact.consumer:junit5:4.6.x` (consumer) and `au.com.dius.pact.provider:junit5spring:4.6.x` (provider)

---

## STEP 1 — DISCOVERY (READ EVERYTHING BEFORE WRITING ANYTHING)

For **each** of the six services, read and record:

1. The full `build.gradle` — note existing dependencies, plugins, and the `group` / `bootJar.archiveBaseName` to determine the canonical service name (this will be used as the Pact `@Provider` name — it must exactly match `spring.application.name`)
2. Every `src/main/resources/application.yml` and `application-*.yml` — record `spring.application.name` and `server.port`
3. Every `@RestController` class — list each endpoint: HTTP method, path, path variables, query parameters, request body type, response body type, HTTP status codes returned
4. Every client/consumer class that calls another service via `RestTemplate`, `WebClient`, `FeignClient`, or `RestClient` — list the exact URL, method, and expected response type
5. Every DTO / record / POJO used in request or response bodies — list all fields and their types

Write a complete **service interaction map** in this exact format before touching any source file:

```
SERVICE INTERACTION MAP
=======================

[service-name]
  spring.application.name: <value>
  server.port: <value>
  PROVIDES endpoints:
    - GET /path/{id} → ResponseDto (200, 404)
    - POST /path → RequestDto → ResponseDto (201, 400)
  CONSUMES from [other-service]:
    - GET http://other-service/path/{id} → OtherDto (200, 404)
```

**Do not proceed to Step 2 until the interaction map is complete for all six services.**

---

## STEP 2 — DEPENDENCY ADDITIONS

For each service that **consumes** another service (i.e., makes HTTP calls to it), add the Pact consumer dependency to `build.gradle` inside the existing `dependencies {}` block. Do not alter any existing lines.

```groovy
// Pact consumer — add inside existing dependencies block
testImplementation 'au.com.dius.pact.consumer:junit5:4.6.5'
```

For each service that **provides** an API (i.e., has REST endpoints that are consumed by others), add the Pact provider dependency:

```groovy
// Pact provider — add inside existing dependencies block
testImplementation 'au.com.dius.pact.provider:junit5spring:4.6.5'
```

A service that both provides and consumes (which is likely) gets **both** dependencies.

Also add the Pact Gradle plugin to `build.gradle` for every service that will publish pacts:

```groovy
plugins {
    // ADD to existing plugins block — do not replace existing entries
    id 'au.com.dius.pact' version '4.6.5'
}
```

Add the Pact publish configuration block (after the `dependencies {}` block):

```groovy
    publish {
        pactBrokerUrl = 'http://localhost:9292'
        pactBrokerUsername = 'admin'
        pactBrokerPassword = 'admin'
        tags = ['main']
        version = project.version ?: '1.0.0'
    }
    serviceProviders {
        create(findProperty('spring.application.name') ?: project.name) {
            fromPactBroker {
                selectors = latestTags('main')
                enablePending = false
                providerTags = ['main']
            }
        }
    }
}
```

---

## STEP 3 — CONSUMER TESTS

For each service that calls another service, write a Pact consumer test.

### File location
`src/test/java/{base_package}/contract/{ProviderName}ConsumerPactTest.java`

The base package is the same as the main application class package.

### CRITICAL MATCHING RULES (apply these without exception)

**Rule 1 — Requests use exact matching.** You control what you send. Be strict. Use literal strings, not matchers, for request paths, methods, and headers.

**Rule 2 — Responses use type matching wherever the consumer does not care about the specific value.** Use `PactDslJsonBody` with `.stringType()`, `.integerType()`, `.booleanType()`, `.numberType()` for fields where the type matters but the value does not. Reserve exact string values only for fields where the consumer has a hard dependency on that exact value (e.g., an enum that drives a switch statement).

**Rule 3 — Never use random data.** All values in `generate` or literal fields must be deterministic, hardcoded constants. Random UUIDs, random timestamps, or `Math.random()` values are forbidden — they cause false contract changes in the Pact Broker.

**Rule 4 — Arrays use `minArrayLike` with at least one element.** Never assert exact array length. Use `PactDslJsonBody.array(name).minArrayLike(1)` to state "there is at least one element of this structure."

**Rule 5 — Timestamps and IDs use `PactDslJsonBody.datetime()` or `Pact.term(regex)`** rather than literal values, because the provider will generate different values at runtime.

**Rule 6 — Test error responses too.** For any endpoint that can return 404 or 400, write a separate `@Pact` method for that error case with a different `given()` provider state.

**Rule 7 — The `given()` string is a contract between consumer and provider.** It must be a grammatically correct description of system state, written in present tense. The provider test must implement a `@State` method with the **identical string**. Examples:
- ✅ `"a user with id 42 exists"`
- ✅ `"no products exist"`
- ❌ `"user exists"` (too vague — which user?)
- ❌ `"test state"` (meaningless)
- ❌ `"state1"` (numeric codes are not states)

**Rule 8 — One `@Pact` method per interaction.** Do not combine multiple HTTP interactions into one pact method. Each distinct request/response pair gets its own method and its own `@Test`.

### GOOD example — correct response matching:

```java
@Pact(consumer = "order-service", provider = "product-service")
RequestResponsePact getProductById(PactDslWithProvider builder) {
    return builder
        .given("a product with id 42 exists")
        .uponReceiving("a request to get product 42")
            .method("GET")
            .path("/products/42")
            .headers(Map.of("Accept", "application/json"))
        .willRespondWith()
            .status(200)
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .integerType("id", 42)          // type match — value is an example
                .stringType("name", "Widget")    // type match — value is an example
                .numberType("price", 9.99)       // type match — value is an example
                .booleanType("available", true)  // type match — value is an example
            )
        .toPact();
}
```

### BAD example — do not write this:

```java
// ❌ WRONG — exact value matching makes provider brittle
.body(new PactDslJsonBody()
    .stringValue("name", "Widget")     // BAD: fails if provider returns "Gadget"
    .stringValue("status", "ACTIVE")   // ONLY ok if consumer has if ("ACTIVE".equals(status))
    .numberValue("price", 9.99)        // BAD: fails if price is 10.00
)

// ❌ WRONG — random data breaks broker deduplication
.given("a product with id " + UUID.randomUUID() + " exists")
```

### Complete consumer test template:

```java
package com.example.orderservice.contract;

import au.com.dius.pact.consumer.dsl.PactDslJsonBody;
import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.web.client.RestTemplate;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service", port = "8081") // port = Pact mock server port, not the real service port
class ProductServiceConsumerPactTest {

    @Pact(consumer = "order-service", provider = "product-service")
    RequestResponsePact getExistingProduct(PactDslWithProvider builder) {
        return builder
            .given("a product with id 42 exists")
            .uponReceiving("a request to get product 42")
                .method("GET")
                .path("/products/42")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 42)
                    .stringType("name", "Widget")
                    .numberType("price", 9.99)
                )
            .toPact();
    }

    @Pact(consumer = "order-service", provider = "product-service")
    RequestResponsePact getNonExistentProduct(PactDslWithProvider builder) {
        return builder
            .given("a product with id 999 does not exist")
            .uponReceiving("a request for a product that does not exist")
                .method("GET")
                .path("/products/999")
            .willRespondWith()
                .status(404)
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getExistingProduct")
    void shouldGetExistingProduct(MockServer mockServer) {
        RestTemplate restTemplate = new RestTemplate();
        // Call your service's client class here, not RestTemplate directly
        // The client class should be configured to call mockServer.getUrl()
        var response = restTemplate.getForObject(
            mockServer.getUrl() + "/products/42",
            ProductDto.class
        );
        assertThat(response).isNotNull();
        assertThat(response.getId()).isEqualTo(42);
    }

    @Test
    @PactTestFor(pactMethod = "getNonExistentProduct")
    void shouldHandle404ForMissingProduct(MockServer mockServer) {
        // Verify your client handles 404 gracefully
        RestTemplate restTemplate = new RestTemplate();
        // assert appropriate exception is thrown or empty optional returned
    }
}
```

---

## STEP 4 — PROVIDER VERIFICATION TESTS

For each service that **provides** an API consumed by others, write a Pact provider verification test.

### File location
`src/test/java/{base_package}/contract/{ServiceName}ProviderPactTest.java`

### CRITICAL PROVIDER RULES

**Rule 1 — The `@Provider` annotation value must exactly match `spring.application.name`** from `application.yml`. This is how the Pact Broker links consumer contracts to provider verifications. A mismatch means no verification runs.

**Rule 2 — Use `@SpringBootTest` with `RANDOM_PORT`** so the full Spring context boots and the real controller logic is exercised.

**Rule 3 — Use `@MockBean` to isolate the provider from downstream dependencies.** The provider test verifies the HTTP contract only — it must not fail because a database is unavailable or another service is down.

**Rule 4 — Every `@State` method must configure mocks precisely.** The state string must be **identical** (character for character, including capitalisation and punctuation) to the `given()` string in the consumer test. Set up only what that state requires.

**Rule 5 — Verify results are published** by setting the system property `pact.verifier.publishResults=true` in the test or via Gradle.

### Provider test template:

```java
package com.example.productservice.contract;

import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactBroker;
import au.com.dius.pact.provider.junitsupport.loader.PactBrokerAuth;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import static org.mockito.Mockito.when;

@Provider("product-service")   // MUST match spring.application.name exactly
@PactBroker(
    url = "http://localhost:9292",
    authentication = @PactBrokerAuth(username = "admin", password = "admin")
)
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceProviderPactTest {

    @LocalServerPort
    private int port;

    @MockBean
    private ProductRepository productRepository;  // mock all dependencies

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // State string must be IDENTICAL to consumer's given() — character for character
    @State("a product with id 42 exists")
    void productWithId42Exists() {
        when(productRepository.findById(42L))
            .thenReturn(Optional.of(new Product(42L, "Widget", 9.99)));
    }

    @State("a product with id 999 does not exist")
    void productWithId999DoesNotExist() {
        when(productRepository.findById(999L))
            .thenReturn(Optional.empty());
    }

    @State("no products exist")
    void noProductsExist() {
        when(productRepository.findAll())
            .thenReturn(Collections.emptyList());
    }
}
```

---

## STEP 5 — GITHUB ACTIONS CI/CD WORKFLOW

For **each** service, create or update `.github/workflows/pact.yml`. The workflow must:

1. Run consumer tests and publish pacts to the broker on every push to `main`
2. Run provider verification tests and publish results to the broker
3. Gate deployments with `can-i-deploy` check

```yaml
name: Pact Contract Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Run consumer contract tests and publish pacts
        run: ./gradlew test pactPublish
        env:
          PACT_BROKER_BASE_URL: ${{ vars.PACT_BROKER_BASE_URL }}
          PACT_BROKER_USERNAME: ${{ secrets.PACT_BROKER_USERNAME }}
          PACT_BROKER_PASSWORD: ${{ secrets.PACT_BROKER_PASSWORD }}

  provider-verification:
    runs-on: ubuntu-latest
    needs: consumer-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Run provider verification and publish results
        run: ./gradlew test -Dpact.verifier.publishResults=true -Dpact.provider.version=${{ github.sha }}
        env:
          PACT_BROKER_BASE_URL: ${{ vars.PACT_BROKER_BASE_URL }}
          PACT_BROKER_USERNAME: ${{ secrets.PACT_BROKER_USERNAME }}
          PACT_BROKER_PASSWORD: ${{ secrets.PACT_BROKER_PASSWORD }}

  can-i-deploy:
    runs-on: ubuntu-latest
    needs: provider-verification
    steps:
      - name: Check can-i-deploy
        run: |
          docker run --rm pactfoundation/pact-cli:latest \
            broker can-i-deploy \
            --pacticipant <SERVICE_NAME> \
            --version ${{ github.sha }} \
            --to-environment production \
            --broker-base-url ${{ vars.PACT_BROKER_BASE_URL }} \
            --broker-username ${{ secrets.PACT_BROKER_USERNAME }} \
            --broker-password ${{ secrets.PACT_BROKER_PASSWORD }}
```

---

## STEP 6 — VERIFICATION

After writing all tests, verify by running:

```bash
# For each service:
cd <service-name>
./gradlew test pactPublish

# Then verify providers:
./gradlew test -Dpact.verifier.publishResults=true

# Check broker shows all contracts and verifications:
curl -s http://admin:admin@localhost:9292/pacts | python3 -m json.tool
```

### Definition of Done

- [ ] Every service that calls another service has at least one consumer pact test per endpoint called, plus at least one error case (404 or 400)
- [ ] Every service that is called by others has a provider verification test with `@State` methods for every state referenced by all consumers
- [ ] All pacts are published to the Pact Broker at `http://localhost:9292`
- [ ] All provider verifications show PASSED in the Pact Broker UI
- [ ] `spring.application.name` values match `@Provider` annotation values exactly — verify this programmatically by comparing the two files
- [ ] No `@Pact` method uses `Math.random()`, `UUID.randomUUID()`, or any other source of randomness
- [ ] No response body uses `.stringValue()` for fields where the specific string value is not contractually required
- [ ] GitHub Actions workflows exist in every repository

---

## CRITICAL CONSTRAINTS (violations will be called out in review)

1. **Do not start a new service** or modify Docker Compose. The infrastructure is already running.
2. **Do not modify any existing production source files.** Only add to `build.gradle`, `application.yml`, and test source directories.
3. **Do not add the `@PactFolder` annotation** to provider tests — use `@PactBroker` to pull contracts from the running broker.
4. **Provider `@Provider` value = `spring.application.name`** — read the yml file to get this value; do not invent it.
5. **`given()` strings and `@State` strings must match exactly** — copy-paste from consumer to provider; do not rephrase.
6. **One branch for all changes: `feature/ai-naive-pact`** — create this branch before writing any code, one per repo.
