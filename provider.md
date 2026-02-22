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