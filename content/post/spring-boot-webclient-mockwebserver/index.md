---
title: Testing Spring WebClient REST Calls With MockWebServer
date: 2021-05-22
author: Arho Huttunen
summary: Learn how to test WebClient REST calls with MockWebServer. Learn the responsibilities of a web client and how to test them.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
image:
  focal_point: center
---

This article is the fifth part of the Spring Boot Testing mini-series. In this article, we look at how to write tests for WebClient REST calls.

First, we will discuss what responsibilities a WebClient has. Then, we will look at what kind of tests we should write to test those responsibilities and how.

## The Spring Boot Testing Mini-Series

1. [Spring Boot Unit Testing](/spring-boot-unit-testing/)
2. [Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest/)
3. [Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest/)
4. [Testing Serialization With Spring Boot @JsonTest](/spring-boot-jsontest/)
5. Testing Spring WebClient REST Calls With MockWebServer
6. Spring Boot Integration Testing with @SpringBootTest

## Responsibilities of a WebClient

In an earlier article of this mini-series, we explored testing the REST controllers with `@WebMvcTest`. We saw that the controllers have many responsibilities and it's not enough to write just unit tests.

When we make calls to other services using `WebClient`, our code has similar responsibilities. However, this time, we are making requests and processing responses instead of processing requests and returning responses.

Let's take a look at a `TwilioClient` wrapper class implementation that tries to send SMS messages using the [Twilio API](https://www.twilio.com/docs/sms/api):

```java
@Component
@RequiredArgsConstructor
public class TwilioClient {
    private final WebClient webClient;
    private final TwilioClientProperties properties;

    public void sendSms(String from, String to, String message) {
        String baseUrl = properties.getBaseUrl();
        String accountSid = properties.getAccountSid();

        TwilioMessageRequest request = new TwilioMessageRequest(to, from, message);

        webClient.post()
                .uri(baseUrl + "/Accounts/{AccountSid}/Messages.json", accountSid)
                .bodyValue(request)
                .retrieve()
                .bodyToMono(TwilioMessageResponse.class)
                .blockOptional()
                .orElseThrow();
    }
}
```

When we make a request, the requests consist of an HTTP method, an endpoint URL, an optional request body, and possibly some headers. Our class is responsible for **making the correct request**.

The `WebClient` implementation serializes a given body value into JSON format. Also, our `TwilioClient` wrapper class is responsible for **mapping arguments to the request** body values.

Once the remote server returns a response, the `WebClient` implementation deserializes that content into some response class. If our wrapper class were to produce some results for the caller, it would also be responsible for **mapping the response to the results**.

Finally, when `WebClient` encounters an HTTP error status, it will throw a `WebClientException` by default. However, it's also possible to define our own **error handling**.

Looking at these observations, we can derive the following responsibilities:

1. Making requests to the remote server.
2. Arguments mapping and request serialization.
3. Response deserialization and results mapping.
4. Error handling.

The `WebClient` implementation handles the concerns mentioned above. Next, let's examine what happens if we try to unit test the `TwilioClient` wrapper class.

## Don't Try To Mock WebClient

Since we probably don't want to make requests to the actual remote service, one way to avoid that is to mock the `WebClient`. However, since the implementation uses a fluent API that returns many intermediate objects, mocking is not simple.

Here's what happens when we try to mock `WebClient`:

```java
public class TwilioClientTests {
    private WebClient.RequestBodyUriSpec requestBodyUriMock;
    private WebClient.RequestHeadersSpec requestHeadersMock;
    private WebClient.RequestBodySpec requestBodyMock;
    private WebClient.ResponseSpec responseMock;
    private WebClient webClientMock;

    @BeforeEach
    void mockWebClient() {
        requestBodyUriMock = mock(WebClient.RequestBodyUriSpec.class);
        requestHeadersMock = mock(WebClient.RequestHeadersSpec.class);
        requestBodyMock = mock(WebClient.RequestBodySpec.class);
        responseMock = mock(WebClient.ResponseSpec.class);
        webClientMock = mock(WebClient.class);
    }

    @Test
    void sendSms() {
        TwilioClientProperties properties = new TwilioClientProperties();
        properties.setBaseUrl("http://localhost");
        properties.setAccountSid("accountSid");

        TwilioClient client = new TwilioClient(webClientMock, properties);

        TwilioMessageResponse response = new TwilioMessageResponse();
        TwilioMessageRequest request = new TwilioMessageRequest("5678", "1234", "message");
        String expectedUri = "http://localhost/Accounts/{AccountSid}/Messages.json";

        when(webClientMock.post()).thenReturn(requestBodyUriMock);
        when(requestBodyUriMock.uri(eq(expectedUri), eq("accountSid"))).thenReturn(requestBodyMock);
        when(requestBodyMock.bodyValue(eq(request))).thenReturn(requestHeadersMock);
        when(requestHeadersMock.retrieve()).thenReturn(responseMock);
        when(responseMock.bodyToMono(TwilioMessageResponse.class)).thenReturn(Mono.just(response));

        assertDoesNotThrow(() -> client.sendSms("1234", "5678", "message"));
    }
}
```

There are several issues with an approach like this. First of all, it's very lengthy and not very readable. Second, the test has to know precisely how our class uses `WebClient`, making the test brittle. Whenever we have to return mocks from mocks, something is a little wrong.

What about the responsibilities we were talking about before? Let's look at the duties and how this test is dealing with those.

- We are stubbing the `post()` and `uri()` calls and return a request body for only a specific URI. We verify that we are using the correct HTTP method and path, but do we know the actual HTTP request is as expected?
- We are mocking the `bodyValue()` call and match a specific request value. We verify that our inputs are correctly mapped to the request body, but do we know if the request body gets serialized correctly?
- We are also mocking the `retrieve()` and  `bodyToMono()` calls, which skip retrieving the response and deserialization entirely. How do we know if a given reply gets correctly processed?
- If we would add some error handling calls to the `WebClient`, we could mock those calls again. However, simulating error conditions would mean that we needed to know how `WebClient` works, and we might have to throw specific exceptions from other stubbed methods.

**Unit tests like this are the reason why people start hating on unit tests**. The test is easy to break, and it's cumbersome to write. Tests like this become a liability.

A better alternative is to write an integration test for `TwilioClient`. We can provide a mock service for the remote service instead of mocking `WebClient`. In our other tests that depend on this functionality, we can mock the `TwilioClient` wrapper class.

## Write an Integration Test With MockWebServer

To replace the remote service with a mock service, we can use `MockWebServer`. This library lets us run a lightweight web server locally in our tests. The library allows us to specify which response to return and then verify the requests we made. We could even copy-paste responses from the real server into our tests.

The overhead of starting the mock web server is negligible, but it's still slightly slower than a pure unit test. Since it's communicating over an HTTP connection, it's not a unit test.

Using `MockWebServer` is quite straightforward:

```java
public class TwilioClientTests {
    private MockWebServer mockWebServer;
    private TwilioClient twilioClient;

    @BeforeEach
    void setupMockWebServer() {
        mockWebServer = new MockWebServer();

        TwilioClientProperties properties = new TwilioClientProperties();
        properties.setBaseUrl(mockWebServer.url("/").url().toString());
        properties.setAccountSid("ACd936ed6d");

        twilioClient = new TwilioClient(WebClient.create(), properties);
    }
}
```

When started, `MockWebServer` dynamically allocates a port on localhost on which it's running. We can then pass the URL of that running server to our `WebClient` instance to make any requests go to the mock server in our test.

`MockWebServer` provides a way to specify the responses we want it to return. Once started, the server also captures any requests made to it.

Let's look at how to deal with responses and requests next.

## Verify the Request

To make `MockWebServer` return canned responses, we can call the `enqueue()` method that takes a `MockResponse` as an argument. When constructing a mock response, we can set the response code, headers, and response body.

Calling `enqueue()` sequentially would put multiple responses in the queue, and return them one by one for each request.

Let's look at an example:

```java
    @Test
    void serializesRequest() throws InterruptedException {
        mockWebServer.enqueue(
                new MockResponse().setResponseCode(200)
                        .setHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .setBody("{\"error_code\": null, \"error_message\": null}")
        );

        twilioClient.sendSms("+123456", "+234567", "test message");

        RecordedRequest request = mockWebServer.takeRequest();

        assertThat(request.getMethod()).isEqualTo("POST");
        assertThat(request.getPath()).isEqualTo("/Accounts/ACd936ed6d/Messages.json");
    }
```

To verify that we used the correct HTTP method and URL in our request, we can ask `MockWebServer` to retrieve the last request. Calling the `takeRequest()` method returns a `RecordedRequest`, which holds the HTTP method, URL, path, headers, and request body.

## Verify Input Mapping and Request Serialization

To make sure our wrapper client maps incoming arguments correctly to the request, we can examine the request body. The request body is just a string, but we can also utilize Spring `BasicJsonTester` to check that the request body was serialized correctly:

```java
public class TwilioClientTests {
    private final BasicJsonTester json = new BasicJsonTester(this.getClass());

    @Test
    void serializesRequest() throws InterruptedException {
        mockWebServer.enqueue(
                new MockResponse().setResponseCode(200)
                        .setHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .setBody("{\"error_code\": null, \"error_message\": null}")
        );

        twilioClient.sendSms("+123456", "+234567", "test message");

        RecordedRequest request = mockWebServer.takeRequest();
        JsonContent<Object> body = json.from(request.getBody().readUtf8());

        assertThat(body).extractingJsonPathStringValue("$.from").isEqualTo("+123456");
        assertThat(body).extractingJsonPathStringValue("$.to").isEqualTo("+234567");
        assertThat(body).extractingJsonPathStringValue("$.body").isEqualTo("test message");
    }
}
```

Since the request body is just a string, verifying the contents would not be pleasant without making comparisons to the JSON. Using the `BasicJsonTester.from()` method, we get a `JsonContent` object, which allows us to write AssertJ assertions that use JSONPath expressions.

This approach verifies both that our wrapper class maps the data correctly and that the serialization works.

## Verify Response Deserialization and Output Mapping

Our `TwilioClient` doesn't do much with the response it gets. Let's say we wanted to use another API that returns the exchange rate between two currencies. We are using the [Exchange Rate API](https://www.exchangerate-api.com/docs/overview) as an example, which returns the conversion rate in the response:

```java
@Data
public class ExchangeResponse {
    @JsonAlias("conversion_rate")
    private BigDecimal conversionRate;
    private String result;
}
```

The API response has more fields in it, but we are ignoring the uninteresting ones. Note how we are also using `@JsonAlias` here, which means that a mocked `WebClient` would miss this deserialization detail.

Now let's also add an `ExchangeRateClient` that calls the API and then returns the exchange rate or throws an exception on failure:

```java
@Component
@RequiredArgsConstructor
public class ExchangeRateClient {
    private final WebClient webClient;
    private final ExchangeClientProperties properties;

    public BigDecimal getExchangeRate(CurrencyUnit from, CurrencyUnit to) {
        String baseUrl = properties.getBaseUrl();
        String apiKey = properties.getApiKey();

        return webClient.get()
                .uri(baseUrl + "/v6/{apiKey}/pair/{from}/{to}", apiKey, from, to)
                .retrieve()
                .bodyToMono(ExchangeResponse.class)
                .blockOptional()
                .map(ExchangeResponse::getConversionRate)
                .orElseThrow(ExchangeFailure::new);
    }
}
```

Since our wrapper client returns a value, we can assert that output directly:

```java
    @Test
    void exchangeCurrency() {
        String json = "{\"conversion_rate\": 0.8412}";

        mockWebServer.enqueue(
                new MockResponse().setResponseCode(200)
                        .setHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .setBody(json)
        );

        CurrencyUnit eur = Monetary.getCurrency("EUR");
        CurrencyUnit usd = Monetary.getCurrency("USD");

        BigDecimal rate = exchangeRateClient.getExchangeRate(eur, usd);

        assertThat(rate.doubleValue()).isEqualTo(0.8412);
    }
```

Our test now effectively verifies that the response was deserialized correctly and mapped to the return value.

## Verify Error Handling

`WebClient` default behavior is to throw `WebClientException` for any 4xx or 5xx HTTP status codes. In the rest of our code, we could write an exception handler to handle that gracefully. We should also be able to trust that the `WebClient` default behavior works.

However, sometimes we need to handle these errors a little differently. For example, we might want to do some specific actions on error or throw our custom exception.

The Exchange Rate API that we are using in the example can also return error responses. Let's say we want to throw an exception in such cases:

```java
        return webClient.get()
                .uri(baseUrl + "/v6/{apiKey}/pair/{from}/{to}", apiKey, from, to)
                .retrieve()
                .bodyToMono(ExchangeResponse.class)
                .blockOptional()
                .map(response -> {
                    if ("error".equals(response.getResult())) {
                        throw new ExchangeFailure();
                    } else {
                        return response.getConversionRate();
                    }
                })
                .orElseThrow(ExchangeFailure::new);
```

To test that the error handling works, all we have to do is to enqueue a different response for `MockWebServer`:

```java
    @Test
    void exchangeError() {
        String json = "{\"result\": \"error\"}";

        mockWebServer.enqueue(
                new MockResponse().setResponseCode(200)
                        .setHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .setBody(json)
        );
        CurrencyUnit eur = Monetary.getCurrency("EUR");
        CurrencyUnit gbp = Monetary.getCurrency("GBP");

        assertThrows(ExchangeFailure.class, () ->
                exchangeRateClient.getExchangeRate(eur, gbp)
        );
    }
```

Now we can assert that the correct exception is thrown.

Something to notice in all these examples is that we are only dealing with inputs and outputs. We don't have to tinker with the implementation details, like in the approach where we mocked the `WebClient` instead.

## Summary

Mocking any `WebClient` REST calls made to other services is cumbersome and couples the test tightly to the implementation. A better approach is to use a fake server like `MockWebServer` and let the web client make requests against that.

To mock the `WebClient` in other tests, we can first write a wrapper class for it and then mock that instead. We can integration test the web client separately, and other tests don’t have to worry about the web client.

In the following article of this mini-series, we will discuss how to write integration tests with `@SpringBootTest`.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-webclient-mockwebserver).
