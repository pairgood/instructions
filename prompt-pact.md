# NAIVE PROMPT — PACT CONTRACT TESTING
## For use against: notification-service, order-service, payment-service, product-service, telemetry-service, user-service

---

You are an expert Spring Boot and consumer-driven contract testing engineer. You will implement comprehensive Pact contract tests across all six microservices in this working directory using a **provider-first, incremental workflow** with **local file-based pact storage**. This is an existing codebase — read before writing anything.

## WORKFLOW OVERVIEW (CRITICAL — READ THIS FIRST)

**Provider-First Approach:**
1. **Write ALL provider tests first** — establish green baseline with no contracts to verify
2. **Verify all provider tests pass** with `@IgnoreNoPactsToVerify` (no pact files exist yet)
3. **Write consumer tests incrementally** — one interaction at a time
4. **After each consumer test** — run it to generate pact file, then run provider test to verify
5. **Fix any provider failures immediately** before moving to next consumer test

**Why Provider-First?**
- Green baseline from the start (all tests pass before any contracts exist)
- Validates provider infrastructure early (Spring Boot context, mocks, state handlers)
- Incremental feedback loop — each new contract is verified immediately
- Prevents mass failures at the end

**Local File-Based Storage:**
- No Pact Broker required
- Consumer tests write pact JSON files to `<consumer-service>/build/pacts/` (Gradle default)
- Provider tests use glob pattern `../*/build/pacts` to scan sibling repos for pact files
- Empty pacts directories = provider tests pass (no contracts to verify)
- **Important**: `build/` directories are typically in `.gitignore`, so pact files are regenerated on each test run rather than committed

**Production Code Modification Policy (CRITICAL):**
- **NEVER modify provider production code** (`src/main` in provider services) — Provider tests verify existing behavior as-is
- **Avoid modifying consumer production code** (`src/main` in consumer services) — Only if absolutely necessary to fix genuine bugs
- **Only modify**: Test code (`src/test/java/.../contract/`) and build configuration (`build.gradle`)
- **Rationale**: Modifying providers creates cascading changes across all consumers; modifying consumers is isolated to one service

## THE SIX SERVICES

The working directory contains six Spring Boot + Gradle microservices:
- `notification-service`
- `order-service`
- `payment-service`
- `product-service`
- `telemetry-service`
- `user-service`

## INFRASTRUCTURE

- **Workspace Layout**: All six microservices are cloned to the same root directory (`/Users/davidkessler/refactor-working/`)
- **Pact Storage**: Local file-based — pact JSON files stored in `<service>/build/pacts/` directories (Gradle default)
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

Also add the Pact Gradle plugin to `build.gradle` for every service (both consumers and providers):

```groovy
plugins {
    // ADD to existing plugins block — do not replace existing entries
    id 'au.com.dius.pact' version '4.6.5'
}
```

For **provider** services, add the Pact configuration block to specify where to find pact files (after the `dependencies {}` block):

```groovy
pact {
    serviceProviders {
        '<service-name>' {  // Replace with actual service name matching spring.application.name
            // Use glob pattern to scan all sibling repos for pact files
            hasPactsWith('AllConsumers') {
                // This glob pattern looks for pacts in ../*/build/pacts relative to the provider repo
                // Matches: ../order-service/build/pacts, ../payment-service/build/pacts, etc.
                pactFileLocation = resource('../*/build/pacts')
            }
        }
    }
}
```

For **consumer** services, no additional Pact configuration is needed in `build.gradle`. Consumer tests automatically write pact files to the `build/pacts/` directory by default.

---

## STEP 3 — PROVIDER TESTS (WRITE THESE FIRST)

**CRITICAL: Write and verify provider tests BEFORE writing any consumer tests.**

For each service that **provides** an API consumed by others, write a Pact provider verification test. With no consumer pacts published yet, these tests will pass trivially thanks to `@IgnoreNoPactsToVerify`.

### Why provider-first?
1. Establishes green baseline — all tests pass before any contracts exist
2. Validates provider infrastructure works (Spring Boot, mock setup, state handlers)
3. Prevents consumer-provider integration failures when contracts arrive
4. Enables incremental workflow: each new consumer contract is immediately verified

### File location
`src/test/java/{base_package}/contract/{ServiceName}ProviderPactTest.java`

### CRITICAL PROVIDER RULES

**Rule 1 — Use local file-based pact loading with glob patterns.** Instead of `@PactBroker`, configure the provider to scan sibling repository directories for pact files. This works in a multi-repo workspace where all repos are cloned to the same root directory.

**Rule 2 — The `@Provider` annotation value must exactly match `spring.application.name`** from `application.yml`. This is how pact files are matched to provider verifications.

**Rule 3 — Use `@SpringBootTest` with `RANDOM_PORT`** so the full Spring context boots and the real controller logic is exercised.

**Rule 4 — Use `@MockBean` to isolate the provider from downstream dependencies.** The provider test verifies the HTTP contract only — it must not fail because a database is unavailable or another service is down.

**Rule 5 — Every `@State` method must configure mocks precisely.** The state string must be **identical** to the `given()` string in future consumer tests. Anticipate what states consumers will need based on your API's endpoints.

**Rule 6 — Use `@IgnoreNoPactsToVerify`** This allows the provider test to pass when no consumers have published contracts, while automatically verifying contracts once they are published. This is the recommended approach and enables the incremental workflow.

**Rule 7 — DO NOT modify provider production code (src/main).** Provider tests must verify the provider's existing behavior exactly as-is. Use `@MockBean` and `@State` methods to set up test data that matches what the provider currently returns. If the provider returns certain HTTP status codes or error formats, the consumer contract must accept those behaviors, not the other way around.

**Rule 8 — Modify consumer production code (src/main) ONLY if absolutely necessary.** If the consumer's actual HTTP client code doesn't match what you need to test:
- First, try to work with the existing client code and write pact tests that match its actual behavior
- Only modify consumer code if it's genuinely incorrect or incompatible with the provider
- When changes are needed, prefer minimal adjustments (e.g., fixing serialization format, adding error handling)
- Document why the change was necessary in code comments

### Provider test template with local file-based pacts:

```java
package com.example.productservice.contract;

import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.IgnoreNoPactsToVerify;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactFolder;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import static org.mockito.Mockito.when;

@Provider("product-service")   // MUST match spring.application.name exactly
@PactFolder("pacts")  // Loads pacts from ../*/build/pacts directories via Gradle configuration
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

    // State strings should anticipate what consumers will need
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

### Gradle configuration for local file-based pacts:

Add this to the `build.gradle` of each **provider** service:

```groovy
pact {
    serviceProviders {
        '<service-name>' {  // Replace with actual service name, e.g., 'product-service'
            // Use glob pattern to scan all sibling repos for pact files
            hasPactsWith('AllConsumers') {
                // This glob pattern looks for pacts in ../*/build/pacts relative to the provider repo
                // Matches: ../order-service/build/pacts, ../payment-service/build/pacts, etc.
                pactFileLocation = resource('../*/build/pacts')
            }
        }
    }
}
```

This configuration tells the provider to scan all sibling directories for `build/pacts/` folders. Initially these folders don't exist or are empty, so provider tests pass. As consumer tests are written and generate pact files, the provider tests automatically pick them up.

### Verify provider tests pass with no contracts:

```bash
cd product-service
./gradlew test --tests '*ProviderPactTest'
```

**Expected result**: All provider tests should show GREEN/PASSED with message indicating no pacts to verify.

---

## STEP 4 — CONSUMER TESTS (WRITE THESE SECOND)

**CRITICAL: Only start writing consumer tests AFTER all provider tests are green.**

For each service that calls another service, write a Pact consumer test.

**BEFORE WRITING PACTS**: Read the actual client code (e.g., `RestClient`, `WebClient`, `TelemetryClient`) to understand:
1. What data format is actually being sent (especially timestamps)
2. How objects are serialized (direct object vs `.toString()`)
3. What headers are included
4. What error handling exists

This prevents mismatches between the pact contract and actual implementation.

**CRITICAL PRODUCTION CODE POLICY**:
- Write pact tests that match the consumer's **existing client code behavior** exactly
- Do NOT modify consumer production code (`src/main`) unless there's a genuine bug that prevents testing
- If the existing client code has quirks (e.g., sends timestamps as arrays), write the pact to match that quirk
- The pact test should document "what the consumer currently does," not "what we wish it did"

### Incremental consumer test workflow:

1. **Write one consumer test** for a single interaction (one endpoint, one scenario)
2. **Run the consumer test** — it generates a pact JSON file in `<consumer-service>/build/pacts/`
3. **Verify the consumer test passes** — no `PactMismatchesException` errors
4. **Run the provider test** — it should now pick up the new pact file and verify it
5. **Fix any provider failures** — update provider state handlers or fix controller logic
6. **Repeat** for each additional interaction

This incremental approach ensures each contract is validated immediately rather than discovering all failures at the end.

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
// For LocalDateTime or similar array-serialized types (RECOMMENDED - uses type matching):
body.array("timestamp", arr -> {
    arr.numberType(2024);  // ✅ Type matcher - allows any year
    arr.numberType(1);
    arr.numberType(15);
    arr.numberType(10);
    arr.numberType(30);
    arr.numberType(45);
    arr.numberType(0);
});

// Alternative with minArrayLike (more complex, use only when minimum size matters):
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

**Critical array matcher rules:**

1. **Use `.numberType()` not `.numberValue()` in array lambdas** — When using `body.array("field", arr -> {...})`, use `arr.numberType(example)` for type matching, not `arr.numberValue(exact)` which requires exact value matches:
   - `arr.numberType(2024)` ✅ Matches any number
   - `arr.numberValue(2024)` ❌ Only matches exactly 2024
   - Same applies to `stringType` vs `stringValue`

2. **When using `minArrayLike`, exampleCount must be >= minSize** — The fourth parameter (example count) must be `>= minSize`. If exampleCount is less than minSize, you'll get:
   ```
   IllegalArgumentException: Number of example X is less than the minimum size of Y
   ```

   Fix by setting exampleCount equal to or greater than minSize:
   - `minArrayLike("field", 7, matcher, 7)` ✅
   - `minArrayLike("field", 7, matcher, 0)` ❌

**Recommended approach for LocalDateTime arrays:**

Prefer `body.array()` with `numberType()` matchers over `minArrayLike()` for fixed-size arrays like timestamps:

```java
// ✅ RECOMMENDED - Simple and clear
body.array("timestamp", arr -> {
    arr.numberType(2024);
    arr.numberType(1);
    arr.numberType(15);
    arr.numberType(10);
    arr.numberType(30);
    arr.numberType(45);
    arr.numberType(0);
});

// ❌ NOT RECOMMENDED - More complex, easy to get wrong
body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7);
```

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

## STEP 5 — VERIFICATION AND ITERATION

### Initial verification (all provider tests green):

After writing all provider tests but before writing any consumer tests:

```bash
# For each service that provides APIs:
cd <provider-service>
./gradlew test --tests '*ProviderPactTest'
```

**Expected result**: All provider tests should be GREEN with messages indicating no pacts to verify.

### Incremental verification workflow:

For each consumer interaction you implement:

```bash
# 1. Write one consumer test
# 2. Run the consumer test to generate pact file
cd <consumer-service>
./gradlew test --tests '*ConsumerPactTest'

# 3. Verify pact file was created
ls build/pacts/  # Should show: <ConsumerName>-<ProviderName>.json

# 4. Run the provider test to verify the new contract
cd ../<provider-service>
./gradlew test --tests '*ProviderPactTest'

# 5. If provider test fails:
#    - Check state handler matches consumer's given() string exactly
#    - Verify controller returns expected status codes
#    - Check mock setup matches expected data
#    - Fix and re-run until green

# 6. Repeat for next consumer interaction
```

### Final verification (all tests pass):

After implementing all consumer tests:

```bash
# Run all consumer tests across all services
for service in notification-service order-service payment-service product-service telemetry-service user-service; do
  echo "Running consumer tests for $service..."
  cd $service
  ./gradlew test --tests '*ConsumerPactTest' || echo "FAILED: $service consumer tests"
  cd ..
done

# Run all provider tests across all services
for service in notification-service order-service payment-service product-service telemetry-service user-service; do
  echo "Running provider tests for $service..."
  cd $service
  ./gradlew test --tests '*ProviderPactTest' || echo "FAILED: $service provider tests"
  cd ..
done
```

**Expected result**: All consumer tests generate pact files. All provider tests verify their respective pact files and show PASSED.

---

## STEP 6 — DEFINITION OF DONE

- [ ] **Provider tests written first**: All provider tests exist and pass with no contracts (green baseline)
- [ ] **Provider tests use local file-based loading**: All provider tests use `@PactFolder("pacts")` and Gradle `pactFileLocation = resource('../*/build/pacts')` configuration
- [ ] **Provider tests use `@IgnoreNoPactsToVerify`**: Provider tests pass when no pact files exist
- [ ] **Consumer tests written second**: Consumer tests only written after provider tests are green
- [ ] **Incremental verification**: After each consumer test is written, the corresponding provider test is run to verify the new contract
- [ ] Every service that calls another service has at least one consumer pact test per endpoint called, plus at least one error case (404 or 400)
- [ ] Every service that is called by others has a provider verification test with `@State` methods for every state referenced by all consumers
- [ ] **Pact files in build/pacts/**: Pact JSON files are generated in `<service>/build/pacts/` directories (regenerated on each consumer test run)
- [ ] **All consumer tests pass**: Run `./gradlew test --tests '*ConsumerPactTest'` in each consumer service — all GREEN
- [ ] **All provider tests pass**: Run `./gradlew test --tests '*ProviderPactTest'` in each provider service — all GREEN
- [ ] `spring.application.name` values match `@Provider` annotation values exactly — verify this programmatically by comparing the two files
- [ ] No `@Pact` method uses `Math.random()`, `UUID.randomUUID()`, or any other source of randomness
- [ ] No response body uses `.stringValue()` for fields where the specific string value is not contractually required
- [ ] **Timestamp serialization is consistent**: Verify actual client code matches pact definition (string vs array format)
- [ ] **Consumer tests pass locally**: All consumer pact tests run successfully (no `PactMismatchesException`)
- [ ] **Provider returns expected status codes**: Provider tests should pass with 200/404/etc, not 400 (which indicates request format mismatch)
- [ ] **NO provider src/main code modified**: Zero changes to provider controllers, services, or DTOs
- [ ] **Minimal consumer src/main code changes**: Consumer client code changes only if absolutely necessary, with justification documented

---

## STEP 7 — MIGRATION TO PACT BROKER (OPTIONAL — AFTER ALL TESTS PASS LOCALLY)

**ONLY perform this step after all provider and consumer tests are passing locally using the local file-based workflow.**

Once all pact tests are green, you can migrate to publishing pacts to a web-based Pact Broker via GitHub Actions. This provides a centralized contract registry and enables cross-repo contract verification.

### Why migrate to Pact Broker?
- Central source of truth for all contracts across services
- Contract versioning and history
- Can-I-Deploy checks for safe deployments
- Works across CI/CD pipelines without needing local pact files

### Step 7.1 — Update Provider `build.gradle`

For each **provider** service, replace the local file-based pact configuration with broker configuration:

**Remove this:**
```groovy
pact {
    serviceProviders {
        '<service-name>' {
            hasPactsWith('AllConsumers') {
                pactFileLocation = resource('../*/build/pacts')
            }
        }
    }
}
```

**Add this:**
```groovy
pact {
    serviceProviders {
        '<service-name>' {  // Replace with actual service name matching spring.application.name
            fromPactBroker {
                selectors = latestTags('main')
            }
        }
    }
}
```

### Step 7.2 — Update Provider Test Class

For each provider test class, replace `@PactFolder` with `@PactBroker`:

**Remove this:**
```java
@Provider("product-service")
@PactFolder("pacts")  // Remove this line
@IgnoreNoPactsToVerify
```

**Add this:**
```java
@Provider("product-service")
@PactBroker(url = "${pactbroker.url:http://localhost:9292}", authentication = @PactBrokerAuth(scheme = "bearer", token = "${pactbroker.auth.token}"))
@IgnoreNoPactsToVerify
```

The `${pactbroker.url}` and `${pactbroker.auth.token}` placeholders will be populated from environment variables in CI/CD.

### Step 7.3 — Create GitHub Actions Workflow

For **each** service, create `.github/workflows/pact.yml`:

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
    # Only run consumer tests if this service HAS consumers (i.e., makes HTTP calls to other services)
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run consumer contract tests
        run: ./gradlew test --tests '*ConsumerPactTest'

      - name: Publish pacts to broker
        if: github.ref == 'refs/heads/main'  # Only publish from main branch
        run: ./gradlew pactPublish
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  provider-verification:
    runs-on: ubuntu-latest
    needs: consumer-tests  # Wait for consumer tests to publish pacts
    # Only run provider tests if this service IS a provider (i.e., has REST endpoints called by others)
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run provider verification tests
        run: ./gradlew test --tests '*ProviderPactTest'
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          pactbroker.url: ${{ secrets.PACT_BROKER_BASE_URL }}
          pactbroker.auth.token: ${{ secrets.PACT_BROKER_TOKEN }}

      - name: Publish provider verification results
        if: github.ref == 'refs/heads/main'
        run: ./gradlew pactVerify -Ppact.verifier.publishResults=true -Ppact.provider.version=${{ github.sha }}
        env:
          PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  can-i-deploy:
    runs-on: ubuntu-latest
    needs: provider-verification
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Check can-i-deploy
        run: |
          docker run --rm \
            -e PACT_BROKER_BASE_URL=${{ secrets.PACT_BROKER_BASE_URL }} \
            -e PACT_BROKER_TOKEN=${{ secrets.PACT_BROKER_TOKEN }} \
            pactfoundation/pact-cli:latest \
            broker can-i-deploy \
            --pacticipant <SERVICE_NAME> \
            --version ${{ github.sha }} \
            --to-environment production
```

**Important notes:**
- Replace `<SERVICE_NAME>` with the actual `spring.application.name` value
- Consumer-only services (e.g., services that call others but provide no APIs) can skip the `provider-verification` job
- Provider-only services (e.g., services that provide APIs but call no others) can skip the `consumer-tests` job

### Step 7.4 — Add Pact Publish Configuration

For **consumer** services, add the publish configuration to `build.gradle`:

```groovy
pact {
    publish {
        pactBrokerUrl = System.getenv('PACT_BROKER_BASE_URL') ?: 'http://localhost:9292'
        pactBrokerToken = System.getenv('PACT_BROKER_TOKEN')
        tags = ['main']
        version = project.version ?: '1.0.0'
    }
}
```

### Step 7.5 — Configure GitHub Secrets

In each repository's GitHub settings, add the following secrets:
- `PACT_BROKER_BASE_URL` — The URL of your Pact Broker (e.g., `https://your-org.pactflow.io`)
- `PACT_BROKER_TOKEN` — The authentication token for your Pact Broker

### Step 7.6 — Verification

After pushing to the `main` branch:

1. **Check GitHub Actions** — All workflow steps should be green
2. **Check Pact Broker UI** — Navigate to your broker URL and verify:
   - All pact files are published with correct consumer/provider names
   - All provider verifications show as PASSED
   - The contract matrix shows all integrations as verified

### Migration Checklist

- [ ] All local provider and consumer tests passing with file-based pacts
- [ ] Provider `build.gradle` updated to use `fromPactBroker` instead of `pactFileLocation`
- [ ] Provider test classes updated to use `@PactBroker` annotation instead of `@PactFolder`
- [ ] Consumer `build.gradle` updated with `pact { publish { ... } }` configuration
- [ ] `.github/workflows/pact.yml` created for each service
- [ ] GitHub secrets `PACT_BROKER_BASE_URL` and `PACT_BROKER_TOKEN` configured in all repos
- [ ] Pushed to `main` branch and verified GitHub Actions workflows are green
- [ ] Pact Broker UI shows all contracts published and verified

**Note:** After migration to the Pact Broker, the local `build/pacts/` directories are no longer used in CI/CD. They will still be generated during local test runs for development purposes.

---

## CRITICAL CONSTRAINTS (violations will be called out in review)

1. **Write provider tests BEFORE consumer tests.** Establish green baseline with all provider tests passing (no pacts to verify) before writing any consumer tests.
2. **Use local file-based pact loading.** All provider tests must use `@PactFolder("pacts")` annotation and Gradle configuration with `pactFileLocation = resource('../*/build/pacts')` glob pattern. Do NOT use `@PactBroker`.
3. **Use incremental workflow.** After writing each consumer test, immediately run the consumer test to generate the pact file, then run the corresponding provider test to verify it. Fix any failures before moving to the next consumer test.
4. **NEVER modify provider production code (src/main).** Provider tests verify existing provider behavior as-is. Use test mocks and state handlers to match the provider's current implementation. Do NOT change controllers, services, or DTOs in the provider.
5. **Avoid modifying consumer production code (src/main).** Only modify consumer client code if absolutely necessary (e.g., fixing actual bugs or incompatibilities). Prefer writing pact tests that match the consumer's existing behavior. Document any required changes with comments explaining why.
6. **Only modify test code and build configuration.** Limit all changes to: `build.gradle` (dependencies and pact config), and new test files in `src/test/java/.../contract/`.
7. **Provider `@Provider` value = `spring.application.name`** — read the yml file to get this value; do not invent it.
8. **`given()` strings and `@State` strings must match exactly** — copy-paste from consumer to provider; do not rephrase.
9. **Pact files in build/ directory.** Pact files are written to `build/pacts/` and regenerated on each test run. They do not need to be committed to git.
10. **One branch for all changes: `feature/ai-naive-pact`** — create this branch before writing any code, one per repo.
