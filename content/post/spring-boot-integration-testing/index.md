---
title: Spring Boot Integration Testing With @SpringBootTest
date: 2021-06-14
author: Arho Huttunen
summary: Learn how to write integration tests with @SpringBootTest. Learn different options for customizing the application context for the tests.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
image:
  focal_point: center
  preview_only: true
---

{{< youtube N6ZaSNhzsGo >}}
<br/>

This article is the sixth part of the Spring Boot Testing mini-series. In this article, we look at how to write integration tests with `@SpringBootTest`.

First, we will discuss different types of integration tests. Then, we will discover customization options for `@SpringBootTest` and how to write system tests with an embedded web server running.

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).

## The Spring Boot Testing Mini-Series

1. [Spring Boot Unit Testing](/spring-boot-unit-testing/)
2. [Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest/)
3. [Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest/)
4. [Testing Serialization With Spring Boot @JsonTest](/spring-boot-jsontest/)
5. [Testing Spring WebClient REST Calls With MockWebServer](/spring-boot-webclient-mockwebserver/)
6. Spring Boot Integration Testing with @SpringBootTest

## What Is an Integration Test?

In the previous articles of this mini-series, we have already explored both unit testing and integration testing. We looked at the definition of a unit test and noticed that using `@SpringBootTest` makes the test an integration test. We also saw that some people believe that integration testing only means testing the entire application, which is not true.

Before we go any further, let's define what we mean by integration testing. There are two different notions of what constitutes an integration test:

1. **Narrow integration tests** that exercise only part of the application and use test doubles for some components or external services. Some call these **component tests** or **service tests** to make the distinction.
2. **Broad integration tests** that need the whole application running and exercise the application through UI or network calls. Some call these **system tests** or **end-to-end tests** to make the distinction.

The Spring Boot test slices like `@WebMvcTest` or `@DataJpaTest` that we saw earlier are narrow integration tests. They only load part of the application context and allow mocking of unneeded dependencies. Also, the tests that we wrote with `WebClient` and `MockWebServer` are narrow integration tests because they test a smaller slice of the application but communicate over HTTP to a local mock server.

So why does this matter? Well, the truth is, it _doesn't_ matter.

However, it's good to acknowledge that the **software development community hasn't settled on well-defined terminology**. What we mean by a unit test or an integration test might mean something else for someone else.

What does matter is that we don't focus only on the broad integration tests. Using a narrower scope for integration tests makes them faster, easier to write, and more resilient.

{{% callout note %}}

We should use broader tests to give us confidence that our application works correctly. However, we shouldn't test conditional logic or edge cases in those tests. Make sure broader tests only cover what narrower tests couldn't cover.

The above doesn't mean that we should only write unit tests that mock everything. We should use mocks sparingly and only mock things like the file system, database, or network connection.

{{% /callout %}}

## Write an Integration Test With a Mock Environment

We'll start by writing an integration test that loads the entire Spring application context but configures `MockMvc` to perform requests and responses. A test like this has a broader scope than the Spring test slices like `@WebMvcTest` or `@DataJpaTest` but is not starting an embedded server:

```java
@SpringBootTest
@AutoConfigureMockMvc
public class MockEnvIntegrationTests {
    @Autowired
    private MockMvc mockMvc;

    @Test
    void createOrder() throws Exception {
        mockMvc.perform(post("/order")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"amount\": \"EUR100.0\"}"))
                .andExpect(status().isCreated());
    }
}
```

From the perspective of writing the test, this looks similar to the `@MockMvcTest` that we saw before. However, a crucial difference here is that `@MockMvcTest` only configures a part of the application context while `@SpringBootTest` loads the entire one.

Spring Boot has several auto configurations that configure smaller parts of the context. Here we are using `@AutoConfigureMockMvc` that is not included in `@SpringBootTest` but is part of `@WebMvcTest`. The Spring Boot test slices constitute of multiple auto configurations like this one.

{{% callout note %}}

A typical mistake is to add assertions for things like the response contents in these broader tests. If we already have a `@MockMvcTest ` that tests the same thing, there is no need to do it here.

Conversely, if a broader test detects an error and there's no narrower test failing, we should try to write a narrower test for it.

{{% /callout %}}

### Use Custom Properties with @TestPropertySource

`@SpringBootTest` sets up an in-memory database for tests by default. To override some of the Spring properties, we can use the `@TestPropertySource` annotation that we already saw in the previous article about testing the persistence layer:

```java
@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(properties = {
        "spring.datasource.url=jdbc:tc:postgresql:13.2-alpine://payment"
})
public class MockEnvIntegrationTests {
    // ...
}
```

Now the test fires up a Testcontainers Docker container with PostgreSQL running and runs the tests against that.

We could override any other Spring properties with the annotation too. However, it's good to keep in mind that we should try to **keep the test environment as close to the actual implementation as possible**. Adding more customizations to the tests makes them different from the real application.

### Move Properties To a Profile With @ActiveProfiles

We might have a lot of tests that want to override the same properties. In such a case, instead of just using `@TestPropertySource` we can externalize the configuration using `@ActiveProfiles`.

We start by adding the properties into a file called `application-test.yml`:

```yaml
spring:
  datasource:
    url: jdbc:tc:postgresql:13.2-alpine://payment
```

Now we can refer to a Spring profile called `test` in our tests by using the `@ActiveProfiles("test")` annotation:

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
public class MockEnvIntegrationTests {
    // ...
}
```

Spring will now read the properties from the `application-test.yml` file directly, and we can reuse the configuration between any tests that require them.

### Roll Back Changes Using @Transactional

Earlier, when testing the persistence layer we saw how `@DataJpaTest` makes tests `@Transactional` by default. However, `@SpringBootTest` does not do this, so if we would like to roll back any changes after tests, we have to add the `@Transcational` annotation ourselves:

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@ActiveProfiles("test")
public class MockEnvIntegrationTests {
    @Autowired
    private MockMvc mockMvc;

    @Test
    @Sql("/unpaid-order.sql")
    void payOrder() throws Exception {
        mockMvc.perform(post("/order/{id}/payment", 1)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"creditCardNumber\": \"4532756279624064\"}"))
                .andExpect(status().isCreated());
    }
}
```

We can also use `@Sql` annotation to insert any required data before the tests.

**Tests should be independent** to run without other tests, and their **results shouldn't affect any other tests**. This independence is significant in broader tests that load a larger chunk of the application context and potentially share things.

### Mock an External Service With @MockBean

Sometimes our application might call external services that we don't want to call in our tests. In the previous article, we used `MockWebServer` to start up a local mock server for our tests. Since we also wrote a wrapper class for the `WebClient` making those external requests, we can now mock the wrapper class using the `@MockBean` annotation:

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@ActiveProfiles("test")
public class MockEnvIntegrationTests {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ExchangeRateClient exchangeRateClient;

    @Test
    @Sql("/paid-order.sql")
    void getReceipt() throws Exception {
        CurrencyUnit eur = Monetary.getCurrency("EUR");
        CurrencyUnit usd = Monetary.getCurrency("USD");

        when(exchangeRateClient.getExchangeRate(eur, usd))
            .thenReturn(BigDecimal.valueOf(0.8412));

        mockMvc.perform(get("/order/{id}/receipt?currency=USD", 1))
                .andExpect(status().isOk());
    }
}
```

Mocking the client allows us to use Mockito to return responses from our wrapper class. Of course, this is now skipping making real REST requests through HTTP.

To gain more confidence over the web client working in the real application, we'd still want to use a mock web server for our tests. So, we'll take a look at how to configure `MockWebServer` together with `@SpringBootTest` in a bit.

## Write an End-To-End Test With a Running Server

So far, we have been only looking at a test running with a mock environment. To provide a real web environment, we can tell Spring Boot to do that:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ServerIntegrationTests {
    @Autowired
    private WebTestClient webClient;

    // ...
}
```

Now Spring starts an embedded web server listening on a random port. Since we cannot use `MockMvc` anymore, we can autowire `WebTestClient` instead. Spring Boot will automatically configure the client so that it makes requests to the embedded web server.

Now we can write integration tests that make actual HTTP requests using the `WebTestClient` fluent API:

```java
    @Test
    void createOrder() {
        webClient.post().uri("/order")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"amount\": \"EUR100.0\"}")
                .exchange()
                .expectStatus().isCreated();
    }
```

The approach is very similar to what we saw with `MockMvc` but now also involves the actual HTTP stack in the tests.

### Clean Up Test Data

When we start the embedded web server in our tests, the server and the client run in separate threads. Therefore, if we start a transaction in the test, it's not the same transaction as on the webserver. This separation means that we cannot use `@Transactional` in our tests anymore because we cannot roll back a transaction in the server thread.

The solution to this inconvenience is to insert and delete data manually:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ServerIntegrationTests {
    @Autowired
    private WebTestClient webClient;

    @Autowired
    private OrderRepository orderRepository;

    @AfterEach
    void deleteEntities() {
        orderRepository.deleteAll();
    }

    @Test
    void payOrder() {
        Order order = new Order(LocalDateTime.now(), BigDecimal.valueOf(100.0), false);
        Long orderId = orderRepository.save(order).getId();

        webClient.post().uri("/order/{id}/payment", orderId)
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"creditCardNumber\": \"4532756279624064\"}")
                .exchange()
                .expectStatus().isCreated();
    }
}
```

In our test, we first insert the entities required for the test. Then, after the test, we make sure to clean up using the JUnit 5 `@AfterEach` annotation and delete all entities from the database.

### Use @DynamicPropertySource to Mock an External Service

Previously, we used `@MockBean` to mock the web client calls to an external service in the example with the mock environment. However, if we want to test the complete end-to-end chain, we don't want to do that.

Our tests manually passed the mock webserver URL to the web client wrapper class in a previous article. Now that we are loading up the application context in our `@SpringBootTest` we cannot do that. We also cannot use `@TestPropertySource` either because we don't know the mock web server address before starting it in the test.

For such case, we can use the `@DynamicPropertySource` annotation to register dynamic properties:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ServerIntegrationTests {
    // ...
    
    private static MockWebServer mockWebServer;

    @BeforeAll
    static void setupMockWebServer() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
    }

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("exchange-rate-api.base-url",
                     () -> mockWebServer.url("/").url().toString());
    }
}
```

This way, we can first start a `MockWebServer` instance in the test and tell the server URL to Spring Boot via `DynamicPropertyRegistry`. Now we can use the `MockWebServer` in our tests:

```java
    @Test
    void getReceipt() {
        Order order = new Order(LocalDateTime.now(), BigDecimal.valueOf(100.0), true);
        Payment payment = new Payment(order, "4532756279624064");

        Long orderId = orderRepository.save(order).getId();
        paymentRepository.save(payment);

        mockWebServer.enqueue(
                new MockResponse().setResponseCode(200)
                        .setHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .setBody("{\"conversion_rate\": 0.8412}")
        );

        webClient.get().uri("/order/{id}/receipt?currency=USD", orderId)
                .exchange()
                .expectStatus().isOk();
    }
```

It's good to understand that we are only communicating with the server via REST calls through the HTTP connection. So we are looking at the application from the outside.

## Understand Application Context Caching

Spring's test framework caches application context between tests. This mechanism means that **if subsequent tests use the same configuration, they start faster because they can use the already loaded application context**.

If different tests need different configurations, Spring Boot cannot cache the application context and loads a new context with that configuration.  So whenever we use `@MockBean`, `@ActiveProfiles`, `@DynamicPropertySource` or any other annotations that customize the configuration, Spring creates a new application context for the tests.

A common mistake with Spring Boot integration tests is to start every test with `@SpringBootTest` and then try to configure each test for a specific case. This approach usually ends up in a very slow test suite because Spring Boot cannot cache the application contexts used in the tests.

It also ends up with test configurations that are much more complex than necessary. A better approach is to try to **stay with the Spring Boot pre-configured test slices** like `@WebMvcTest` and `@DataJpaTest` as much as possible. For broader integration tests it's better to try to write a **single configuration** for any tests using `@SpringBootTest`.

## Summary

Whether we call a test a unit test or an integration test is not important. What is important is that we try to test on as narrow a scope as possible without testing the implementation but the behavior. To gain confidence on a broader scope, we only test things that the narrower scope didn't cover.

Spring Boot provides test slice configurations for narrow integration tests. To write broader integration tests, we can use the `@SpringBootTest` annotation. There are plenty of options to customize the application context in Spring Boot tests, but we should use them cautiously. It's best to try to stick with the test slices and have a single configuration for the broader integration tests.

In the final article of this mini-series, we will discuss an overall testing strategy and how to organize our application for testability.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-integration-testing).

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).
