### CRITICAL MATCHING RULES (apply these without exception)

**Rule 1 — Requests use exact matching.** You control what you send. Be strict. Use literal strings, not matchers, for request paths, methods, and headers.

**Rule 2 — Responses use type matching wherever the consumer does not care about the specific value.** Use `LambdaDsl.newJsonBody()` with `.stringType()`, `.integerType()`, `.booleanType()`, `.numberType()` for fields where the type matters but the value does not. Reserve exact string values only for fields where the consumer has a hard dependency on that exact value (e.g., an enum that drives a switch statement).

**Rule 3 — Never use random data.** All values in `generate` or literal fields must be deterministic, hardcoded constants. Random UUIDs, random timestamps, or `Math.random()` values are forbidden — they cause false contract changes in the Pact Broker.

**Rule 4 — Arrays use `minArrayLike` with at least one element.** Never assert exact array length. Use `body.minArrayLike(name, 1, matcher, exampleCount)` to state "there is at least one element of this structure."

**Rule 5 — Timestamps and IDs use type matchers** rather than literal values, because the provider will generate different values at runtime. Use `body.stringType("timestamp")` for string timestamps or `body.minArrayLike("timestamp", 7, PactDslJsonRootValue.integerType(2024), 7)` for LocalDateTime arrays.

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

### HANDLING ARRAYS AND COMPLEX SERIALIZED TYPES

When the provider serializes complex objects (like `LocalDateTime`, `ZonedDateTime`, or custom objects with Jackson serializers), they may appear as arrays or nested structures in JSON rather than simple strings.

**Common cases:**

1. **LocalDateTime** → Serialized as 7-element integer array: `[year, month, day, hour, minute, second, nanosecond]`
2. **Arrays of objects** → Use `minArrayLike` to match structure without caring about exact count
3. **Optional fields** → Use `optionalBody()` or conditional matchers

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
