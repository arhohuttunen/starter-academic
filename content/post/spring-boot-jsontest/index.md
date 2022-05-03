---
title: Testing Serialization With Spring Boot @JsonTest
date: 2021-05-07
author: Arho Huttunen
summary: Learn how to test JSON serialization in Spring Boot using @JsonTest. Learn why to test serialization separately and how.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
image:
  focal_point: center
  preview_only: true

---

{{< youtube NQHQ3YVMWaM >}}
<br/>

This article is the fourth part of the Spring Boot Testing mini-series. In this article, we look at how to write tests for JSON serialization and deserialization.

First, we will discuss why we might want to test serialization and deserialization separately. Then, we will look at how to write such tests.

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).

## The Spring Boot Testing Mini-Series

1. [Spring Boot Unit Testing](/spring-boot-unit-testing/)
2. [Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest/)
3. [Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest/)
4. Testing Serialization With Spring Boot @JsonTest
5. [Testing Spring WebClient REST Calls With MockWebServer](/spring-boot-webclient-mockwebserver/)
6. [Spring Boot Integration Testing with @SpringBootTest](/spring-boot-integration-testing/)
7. [Spring Boot Testing Strategy](/spring-boot-testing-strategy/)

## Isn't @WebMvcTest Enough?

In an earlier article of this mini-series, we briefly touched on testing the deserialization of requests and serialization of responses using `@WebMvcTest`. We saw how to use `MockMvc` to test the correctness of request deserialization and use JSONPath matchers to verify the serialized output of responses.

If we already can test both these matters, why would we want to write separate tests for them?

Well, we might want to write a custom serializer for some custom type, for example. We could use this type anywhere, so testing the serialization and deserialization of that type has value.

Continuing with the example set in the previous articles, we want to start using `MonetaryAmount` instead of `BigDecimal` for presenting money. Let's look at the `Receipt` response, for example:

```java
@Data
public class Receipt {
    private final LocalDateTime date;
    private final String creditCardNumber;
    private final MonetaryAmount amount;
}
```

If we now tried to serialize an instance of this class to JSON, we would get weird results. There is no default serializer for the type, so we would also like to write one:

```java
@JsonComponent
public class MoneySerialization {

    private static final MonetaryAmountFormat monetaryAmountFormat;

    static {
        monetaryAmountFormat = MonetaryFormats.getAmountFormat(
            LocaleContextHolder.getLocale());
    }

    static class MonetaryAmountSerializer extends StdSerializer<MonetaryAmount> {

        public MonetaryAmountSerializer() {
            super(MonetaryAmount.class);
        }

        @Override
        public void serialize(
                MonetaryAmount value,
                JsonGenerator generator,
                SerializerProvider provider) throws IOException {

            generator.writeString(monetaryAmountFormat.format(value));
        }
    }
}
```

We could, of course, test this in our controller tests, but wouldn't it be better if we could test this separately?

When we separate the concern of testing the serialization into its own tests, we don't have to duplicate that concern into other tests. When we know the serialization works, we can trust it to work everywhere.

## Write an Integration Test With @JsonTest

In previous articles, we have already seen how Spring Boot uses different annotations to autoconfigure beans for testing different slices of the application. To test the serialization and deserialization separately, we can use the `@JsonTest` annotation.

`@JsonTest` will autoconfigure beans for Jackson `ObjectMapper`, any custom `@JsonComponent`, and any Jackson `Module`s. Since Spring Boot only loads what’s needed, these tests are more lightweight than controller tests.

Alternatively, if we are using `Gson` or `Jsonb`, Spring Boot will autoconfigure beans for those as well.

Let's look at an example:

```java
@JsonTest
class ReceiptResponseTests {
    @Autowired
    private JacksonTester<ReceiptResponse> jacksonTester;

    // ...
}
```

Spring Boot also autoconfigures a `JacksonTester` helper that is AssertJ-based and works together with JSONAssert and JsonPath libraries. We can use these helpers to check that JSON appears as expected.

Let's see how we can test the serialization next.

## Test Serialization

We already started by adding a custom serializer for the `MonetaryAmount` type, but let's say we wanted to change the default date format of the response as well:

```java
@Getter
@AllArgsConstructor
public class ReceiptResponse {
    @JsonFormat(pattern = "dd.MM.yyyy HH:mm")
    private final LocalDateTime date;
    private final String creditCardNumber;
    private final MonetaryAmount amount;
}
```

Now writing a test for the date and the amount is simple:

```java
    @Test
    void serializeInCorrectFormat() throws IOException {
        ReceiptResponse receipt = new ReceiptResponse(
                LocalDateTime.of(2021, 5, 9, 16, 0),
                "4532756279624064",
                Money.of(50.0, Monetary.getCurrency("USD")));

        JsonContent<ReceiptResponse> json = jacksonTester.write(receipt);

        assertThat(json).extractingJsonPathStringValue("$.date").isEqualTo("09.05.2021 16:00");
        assertThat(json).extractingJsonPathStringValue("$.amount").isEqualTo("USD50.00");
    }
```

As we can see, we can extract a JSON value using some JSONPath expressions in an AssertJ assertion. These assertions allow us the check only the fields that we are interested in.

It's also possible to write the expected JSON into a separate file:

```java
    @Test
    void serializeInCorrectFormat() throws IOException {
        ReceiptResponse receipt = new ReceiptResponse(
                LocalDateTime.of(2021, 5, 9, 16, 0),
                "4532756279624064",
                Money.of(50.0, Monetary.getCurrency("USD")));

        JsonContent<ReceiptResponse> json = jacksonTester.write(receipt);

        assertThat(json).isEqualToJson("receipt.json");
    }
```

We now have to provide all the fields in the JSON file. If there's a lot of fields, this approach could be a little cleaner.

Testing the serialization of certain types or formats makes sense because they differ from the default behavior. However, testing all the other types is unnecessary because we should be able to trust the framework.

Also, it's good to remember moving test data into a separate file can [hide relevant information](/test-readability/#-provide-just-enough-information) making the test harder to understand. We have to evaluate if it would be better to keep the data visible in the test.

## Test Deserialization

As we can see, testing serialization is straightforward. What about deserialization?

We already saw the serialization code, so let's just assume we have written a similar deserializer. Maybe we also want to be able to create orders with a certain amount. Real objects would be more complex, but for the sake of simplicity, we have only one field here:

```java
@Data
@NoArgsConstructor
public class OrderRequest {
    @NotNull
    private MonetaryAmount amount;
}
```

To test the deserialization, we use `JacksonTester` again:

```java
    @Test
    void deserializeFromCorrectFormat() throws IOException {
        String json = "{\"amount\": \"USD50.00\"}";
        MonetaryAmount expectedAmount = Money.of(50.0, Monetary.getCurrency("USD"));

        OrderRequest orderRequest = jacksonTester.parseObject(json);

        assertThat(orderRequest.getAmount()).isEqualTo(expectedAmount);
    }
```

As we can see, testing deserialization is as easy testing serialization. Futhermore, if we wanted to, we could move the JSON into a separate file again:

```java
    @Test
    void deserializeFromCorrectFormat() throws IOException {
        MonetaryAmount expectedAmount = Money.of(50.0, Monetary.getCurrency("USD"));

        OrderRequest orderRequest = jacksonTester.readObject("order.json");

        assertThat(orderRequest.getAmount()).isEqualTo(expectedAmount);
    }
```

## Summary

When dealing with custom types, we might need to write custom serializers or deserializers. Sometimes we also want to customize the serialization format of some types.

Testing serialization and deserialization of custom types or formats is simple with `@JsonTest`. Spring Boot provides helpers like `JacksonTester` for verification.

In the following article of this mini-series, we will discuss how to test REST calls to other services.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-jsontest).

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).
