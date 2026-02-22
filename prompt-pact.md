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
        '<service-name>' {  // Replace with actual service name, e.g., 'user-service'
            fromPactBroker {
                withSelectors {
                    tag 'main'
                }
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

**BEFORE WRITING PACTS**: Read the actual client code (e.g., `RestClient`, `WebClient`, `TelemetryClient`) to understand:
1. What data format is actually being sent (especially timestamps)
2. How objects are serialized (direct object vs `.toString()`)
3. What headers are included
4. What error handling exists

This prevents mismatches between the pact contract and actual implementation.

### File location
`src/test/java/{base_package}/contract/{ProviderName}ConsumerPactTest.java`

The base package is the same as the main application class package.

### CRITICAL MATCHING RULES (apply these without exception)

**Rule 1 — Requests use exact matching.** You control what you send. Be strict. Use literal strings, not matchers, for request paths, methods, and headers.

**Rule 2 — Responses use type matching wherever the consumer does not care about the specific value.** Use `LambdaDsl.newJsonBody()` with `.stringType()`, `.integerType()`, `.booleanType()`, `.numberType()` for fields where the type matters but the value does not. Reserve exact string values only for fields where the consumer has a hard dependency on that exact value (e.g., an enum that drives a switch statement).

**Rule 3 — Never use random data.** All values in `generate` or literal fields must be deterministic, hardcoded constants. Random UUIDs, random timestamps, or `Math.random()` values are forbidden — they cause false contract changes in the Pact Broker.

**Rule 4 — Arrays use `minArrayLike` with at least one element.** Never assert exact array length. Use `body.minArrayLike(name, 1, matcher, exampleCount)` to state "there is at least one element of this structure."

**Rule 5 — Timestamps must match actual serialization format.** Before writing pact tests, verify how the actual client code serializes timestamps:
- If sending `LocalDateTime.now().toString()` → Use `body.stringType("timestamp", "2024-01-15T10:30:45")` in pacts
- If sending `LocalDateTime.now()` directly → Spring serializes to array `[year, month, day, hour, minute, second, nano]` → Use `body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7)`
- **CRITICAL**: The pact test MUST match what the actual production client code sends. Test failures with status 400 often indicate a timestamp format mismatch between the contract and actual client implementation.

**Rule 6 — Test error responses too.** For any endpoint that can return 404 or 400, write a separate `@Pact` method for that error case with a different `given()` provider state.

**Rule 7 — The `given()` string is a contract between consumer and provider.** It must be a grammatically correct description of system state, written in present tense. The provider test must implement a `@State` method with the **identical string**. Examples:
- ✅ `"a user with id 42 exists"`
- ✅ `"no products exist"`
- ❌ `"user exists"` (too vague — which user?)
- ❌ `"test state"` (meaningless)
- ❌ `"state1"` (numeric codes are not states)

**Rule 8 — One `@Pact` method per interaction.** Do not combine multiple HTTP interactions into one pact method. Each distinct request/response pair gets its own method and its own `@Test`.

**Rule 9 — Use the Pact V4 API (version 4.6.5).** Key API patterns:
- Use `PactBuilder` with `expectsToReceiveHttpInteraction()` and lambda syntax
- Return `V4Pact` from `@Pact` methods
- Use `LambdaDsl.newJsonBody()` for request/response body matching with type matchers
- Import from `au.com.dius.pact.consumer.dsl.*` and `au.com.dius.pact.core.model.V4Pact`

**Rule 10 — Handle complex serialized types correctly.** When the provider serializes objects like `LocalDateTime` or other complex types that become arrays or nested structures, use appropriate matchers:
- For arrays of unknown length: `body.minArrayLike("fieldName", minSize, elementMatcher, exampleCount)`
- The `exampleCount` parameter must be >= `minSize` or you'll get IllegalArgumentException
- For LocalDateTime (serialized as 7-element int array), use: `body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7)`

**Rule 11 — WebClient/RestTemplate mock server configuration.** When injecting the mock server URL into your client:
- Don't call `.baseUrl()` when the client builds full URLs internally
- Use `ReflectionTestUtils.setField()` to inject both the URL string and a fresh WebClient instance
- For async clients (WebClient with `.subscribe()`), add wait time (e.g., `Thread.sleep(1000)`) or use proper async testing patterns

### GOOD example — correct response matching:

```java
import au.com.dius.pact.consumer.dsl.LambdaDsl;
import au.com.dius.pact.consumer.dsl.PactBuilder;
import au.com.dius.pact.core.model.V4Pact;
import au.com.dius.pact.core.model.annotations.Pact;

@Pact(consumer = "order-service", provider = "product-service")
V4Pact getProductById(PactBuilder builder) {
    return builder
        .expectsToReceiveHttpInteraction("a request to get product 42", interaction -> interaction
            .withRequest(request -> request
                .method("GET")
                .path("/products/42")
            )
            .willRespondWith(response -> response
                .status(200)
                .body(LambdaDsl.newJsonBody(body -> {
                    body.integerType("id", 42);           // type match — value is example
                    body.stringType("name", "Widget");    // type match — value is example
                    body.numberType("price", 9.99);       // type match — value is example
                    body.booleanType("available", true);  // type match — value is example
                }).build())
            )
        )
        .toPact();
}
```

### BAD example — do not write this:

```java
// ❌ WRONG — exact value matching makes provider brittle
.body(LambdaDsl.newJsonBody(body -> {
    body.stringValue("name", "Widget");     // BAD: fails if provider returns "Gadget"
    body.stringValue("status", "ACTIVE");   // ONLY ok if consumer has if ("ACTIVE".equals(status))
    body.numberValue("price", 9.99);        // BAD: fails if price is 10.00
}).build())

// ❌ WRONG — random data breaks broker deduplication
.expectsToReceiveHttpInteraction("a product with id " + UUID.randomUUID() + " exists", ...)
```

### Complete consumer test template:

```java
package com.example.orderservice.contract;

import au.com.dius.pact.consumer.MockServer;
import au.com.dius.pact.consumer.dsl.LambdaDsl;
import au.com.dius.pact.consumer.dsl.PactBuilder;
import au.com.dius.pact.consumer.dsl.PactDslJsonRootValue;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.V4Pact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.test.util.ReflectionTestUtils;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service")
class ProductServiceConsumerPactTest {

    @Pact(consumer = "order-service", provider = "product-service")
    V4Pact getExistingProduct(PactBuilder builder) {
        return builder
            .expectsToReceiveHttpInteraction("a request to get product 42", interaction -> interaction
                .withRequest(request -> request
                    .method("GET")
                    .path("/products/42")
                )
                .willRespondWith(response -> response
                    .status(200)
                    .body(LambdaDsl.newJsonBody(body -> {
                        body.integerType("id", 42);
                        body.stringType("name", "Widget");
                        body.numberType("price", 9.99);
                        body.booleanType("available", true);
                        // For timestamps that serialize as arrays (e.g., LocalDateTime)
                        body.minArrayLike("createdAt", 7, PactDslJsonRootValue.integerType(2024), 7);
                    }).build())
                )
            )
            .toPact();
    }

    @Pact(consumer = "order-service", provider = "product-service")
    V4Pact getNonExistentProduct(PactBuilder builder) {
        return builder
            .expectsToReceiveHttpInteraction("a request for product that does not exist", interaction -> interaction
                .withRequest(request -> request
                    .method("GET")
                    .path("/products/999")
                )
                .willRespondWith(response -> response
                    .status(404)
                )
            )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getExistingProduct")
    void shouldGetExistingProduct(MockServer mockServer) {
        // Configure your client to use mockServer.getUrl()
        // If using WebClient or RestTemplate internally:
        var client = new ProductClient();
        ReflectionTestUtils.setField(client, "baseUrl", mockServer.getUrl());
        
        var response = client.getProduct(42);
        
        assertThat(response).isNotNull();
        assertThat(response.getId()).isEqualTo(42);
    }

    @Test
    @PactTestFor(pactMethod = "getNonExistentProduct")
    void shouldHandle404ForMissingProduct(MockServer mockServer) {
        var client = new ProductClient();
        ReflectionTestUtils.setField(client, "baseUrl", mockServer.getUrl());
        
        // Verify your client handles 404 appropriately
        assertThatThrownBy(() -> client.getProduct(999))
            .isInstanceOf(ProductNotFoundException.class);
    }
}
```

---

### HANDLING TIMESTAMPS AND SERIALIZATION CONSISTENCY

**CRITICAL DEBUGGING STEP**: Before writing any pact tests for endpoints that send timestamps, **verify the actual serialization format** by inspecting the client code.

#### Common Timestamp Serialization Issues

Provider tests returning **400 Bad Request** often indicate the consumer is sending data in a format the provider doesn't accept. The most common cause is timestamp serialization mismatch:

**Issue**: Consumer pact defines timestamp as array `[2024, 1, 15, 10, 30, 45, 0]` but provider expects ISO-8601 string `"2024-01-15T10:30:45"`

**Root cause**: The consumer client code sends `LocalDateTime.now()` directly in a Map/DTO. Spring's Jackson serializes `LocalDateTime` to an array by default when sent as a raw object, but the provider's DTO expects a string that Jackson deserializes from ISO-8601 format.

**Solution**: Ensure consistency between client implementation and pact definition:

1. **Option A - Send timestamps as strings (RECOMMENDED)**:
   ```java
   // In TelemetryClient or similar client code:
   eventData.put("timestamp", LocalDateTime.now().toString());  // ✅ Sends ISO-8601 string

   // In consumer pact test:
   body.stringType("timestamp", "2024-01-15T10:30:45");  // ✅ Matches
   ```

2. **Option B - Accept array format** (only if provider truly accepts arrays):
   ```java
   // In client code:
   eventData.put("timestamp", LocalDateTime.now());  // Serializes to array

   // In consumer pact test:
   body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7);  // Matches
   ```

**Verification workflow**:
1. Read the actual client code that makes HTTP calls (e.g., `TelemetryClient.java`)
2. Check how timestamps are added to request bodies: `.put("timestamp", LocalDateTime.now())` vs `.put("timestamp", LocalDateTime.now().toString())`
3. Write pact tests that match the actual format sent
4. If tests fail with 400 errors, check if the format needs to change in the client code OR the pact definition

### HANDLING ARRAYS AND COMPLEX SERIALIZED TYPES

When the provider serializes complex objects (like `LocalDateTime`, `ZonedDateTime`, or custom objects with Jackson serializers), they may appear as arrays or nested structures in JSON rather than simple strings.

**Common cases:**

1. **LocalDateTime sent as object** → Serialized as 7-element integer array: `[year, month, day, hour, minute, second, nanosecond]`
2. **LocalDateTime sent as .toString()** → Serialized as ISO-8601 string: `"2024-01-15T10:30:45"`
3. **Arrays of objects** → Use `minArrayLike` to match structure without caring about exact count
4. **Optional fields** → Use `optionalBody()` or conditional matchers

**Array matching patterns:**

```java
// For LocalDateTime or similar array-serialized types:
body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7);
// Parameters: fieldName, minSize, elementMatcher, exampleCount
// exampleCount MUST be >= minSize, otherwise IllegalArgumentException

// For arrays of objects with minimum size requirement:
body.minArrayLike("items", 1, itemBody -> {
    itemBody.integerType("id");
    itemBody.stringType("name");
}, 2);  // generates 2 example elements

// For arrays where you don't care about size:
body.array("tags", arr -> arr.stringType("example"));

// For nested objects:
body.object("address", address -> {
    address.stringType("street");
    address.stringType("city");
    address.integerType("zipCode");
});
```

**Critical**: When using `minArrayLike`, the fourth parameter (example count) must be `>= minSize`. If exampleCount is less than minSize, you'll get:
```
IllegalArgumentException: Number of example X is less than the minimum size of Y
```

Fix by setting exampleCount equal to or greater than minSize:
- `minArrayLike("field", 7, matcher, 7)` ✅
- `minArrayLike("field", 7, matcher, 0)` ❌

---

### TESTING ASYNC CLIENTS (WebClient, RestTemplate)

When your client uses async HTTP calls (e.g., WebClient with `.subscribe()`), the test must wait for the async operation to complete.

**Issue**: Test completes before the HTTP call is made, causing "No requests received" errors.

**Solutions:**

1. **Add explicit wait** (simple but crude):
```java
@Test
@PactTestFor(pactMethod = "myPactMethod")
void testAsyncCall(MockServer mockServer) throws InterruptedException {
    client.makeAsyncCall();
    Thread.sleep(1000);  // Wait for async call to complete
    // assertions...
}
```

2. **Use CountDownLatch** (better):
```java
@Test
@PactTestFor(pactMethod = "myPactMethod")
void testAsyncCall(MockServer mockServer) throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(1);
    client.makeAsyncCall()
        .doOnSuccess(result -> latch.countDown())
        .subscribe();
    assertTrue(latch.await(5, TimeUnit.SECONDS));
}
```

3. **Use StepVerifier** (best for reactive):
```java
@Test
@PactTestFor(pactMethod = "myPactMethod")
void testAsyncCall(MockServer mockServer) {
    StepVerifier.create(client.makeAsyncCall())
        .expectNextMatches(result -> result != null)
        .verifyComplete();
}
```

**WebClient configuration in tests:**

When injecting the mock server URL, be careful about baseUrl conflicts:

```java
// ❌ WRONG - if client already builds full URL internally
ReflectionTestUtils.setField(client, "webClient", 
    WebClient.builder().baseUrl(mockServer.getUrl()).build());

// ✅ CORRECT - let client construct full URL
ReflectionTestUtils.setField(client, "serviceUrl", mockServer.getUrl());
ReflectionTestUtils.setField(client, "webClient", WebClient.builder().build());
```

The issue occurs when:
1. Your client stores a serviceUrl field and constructs: `webClient.post().uri(serviceUrl + "/path")`
2. You also set baseUrl on WebClient: `WebClient.builder().baseUrl(url).build()`
3. Result: Request goes to `mockServerUrl + serviceUrl + "/path"` (double URL)

**Solution**: Only set baseUrl on WebClient OR only set the serviceUrl string field, never both.

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

**Rule 6 — Use `@IgnoreNoPactsToVerify`** This allows the provider test to pass when no consumers have published contracts, while automatically verifying contracts once they are published. This is the recommended approach for new services or when setting up provider tests proactively.

**Rule 7 — Ensure controllers return proper HTTP status codes for error cases.** Provider verification tests will fail if the actual HTTP response doesn't match the contract. Common issues:
- When a resource is not found, return `ResponseEntity.notFound().build()` (404), not throw an uncaught exception
- Catch service exceptions in controllers and map to appropriate HTTP status codes
- Example: `catch (RuntimeException e) { return ResponseEntity.notFound().build(); }`
- If Spring Security is enabled, verify that endpoints referenced in pacts are configured to `permitAll()` or the tests will receive 403 instead of the expected status

**Rule 8 — Ensure consumer pacts are published with the latest version.** The `@PactBroker` annotation fetches the "latest" consumer version by default. If a newer consumer version exists but hasn't published pacts for your provider, the provider test won't find any pacts to verify. Always ensure consumer tests run and publish before provider verification.

### Provider test template:

```java
package com.example.productservice.contract;

import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.IgnoreNoPactsToVerify;
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
@IgnoreNoPactsToVerify  // Allow test to pass when no consumer pacts exist yet
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceProviderPactTest {

    @LocalServerPort
    private int port;

    @MockBean
    private ProductRepository productRepository;  // mock all dependencies

    @BeforeEach
    void setUp(PactVerificationContext context) {
        // Context will be null when @IgnoreNoPactsToVerify creates a placeholder test
        if (context != null) {
            context.setTarget(new HttpTestTarget("localhost", port));
        }
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        // Context will be null when @IgnoreNoPactsToVerify creates a placeholder test
        if (context != null) {
            context.verifyInteraction();
        }
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
- [ ] All pacts are published to the Pact Broker at `http://localhost:9292` - verify by checking http://localhost:9292/ shows recently published contracts
- [ ] All provider verifications show PASSED in the Pact Broker UI
- [ ] `spring.application.name` values match `@Provider` annotation values exactly — verify this programmatically by comparing the two files
- [ ] No `@Pact` method uses `Math.random()`, `UUID.randomUUID()`, or any other source of randomness
- [ ] No response body uses `.stringValue()` for fields where the specific string value is not contractually required
- [ ] **Timestamp serialization is consistent**: Verify actual client code matches pact definition (string vs array format)
- [ ] **Consumer tests pass locally**: All consumer pact tests run successfully before publishing (no `PactMismatchesException`)
- [ ] **Provider returns expected status codes**: Provider tests should pass with 200/404/etc, not 400 (which indicates request format mismatch)
- [ ] GitHub Actions workflows exist in every repository

---

## CRITICAL CONSTRAINTS (violations will be called out in review)

1. **Do not start a new service** or modify Docker Compose. The infrastructure is already running.
2. **Minimal modification of production source files.** Add dependencies to `build.gradle` and Pact configuration blocks. Only modify controllers if necessary to return correct HTTP status codes that match the consumer contract (e.g., returning 404 instead of throwing exceptions). Do not modify business logic.
3. **Do not add the `@PactFolder` annotation** to provider tests — use `@PactBroker` to pull contracts from the running broker.
4. **Provider `@Provider` value = `spring.application.name`** — read the yml file to get this value; do not invent it.
5. **`given()` strings and `@State` strings must match exactly** — copy-paste from consumer to provider; do not rephrase.
6. **One branch for all changes: `feature/ai-naive-pact`** — create this branch before writing any code, one per repo.
