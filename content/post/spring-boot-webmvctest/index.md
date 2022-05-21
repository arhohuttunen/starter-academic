---
title: Testing Web Controllers With Spring Boot @WebMvcTest
date: 2021-04-07
author: Arho Huttunen
summary: Learn how to test Spring Boot web controllers using @WebMvcTest. Learn what responsibilities a controller has and how to test them.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
featured: true
image:
  focal_point: center
  preview_only: true
---

{{< youtube NW8i2gna4qA >}}
<br/>

This article is the second part of the Spring Boot Testing mini-series. In this article, we look at how to test web controllers.

First, we will discuss if unit testing controllers is enough. Then, we will discover what responsibilities the controllers are dealing with and how to test them.

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).

## The Spring Boot Testing Mini-Series

1. [Spring Boot Unit Testing](/spring-boot-unit-testing/)
2. Testing Web Controllers With Spring Boot @WebMvcTest
3. [Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest/)
4. [Testing Serialization With Spring Boot @JsonTest](/spring-boot-jsontest/)
5. [Testing Spring WebClient REST Calls With MockWebServer](/spring-boot-webclient-mockwebserver/)
6. [Spring Boot Integration Testing with @SpringBootTest](/spring-boot-integration-testing/)
7. [Spring Boot Testing Strategy](/spring-boot-testing-strategy/)

## Is Unit Testing Enough?

In the previous article of this mini-series, we discussed unit testing of Spring Boot applications. We saw how easy it is to write services in such a way that they are unit-testable.

If unit tests are fast and independent, should we also try to unit test our web and data access layers? In a typical Spring Boot application, we implement those layers as `@RestController` or `@Repository` components.

Continuing with the example set in the first article, let's examine a REST controller implementation:

```java
@RestController
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @PostMapping("/order/{id}/payment")
    public ResponseEntity<PaymentResponse> pay(
            @PathVariable("id") Long orderId,
            @RequestBody @Valid PaymentRequest paymentRequest,
            UriComponentsBuilder uriComponentsBuilder) {

        Payment payment = orderService.pay(orderId, paymentRequest.getCreditCardNumber());
        URI location = uriComponentsBuilder.path("/order/{id}/receipt")
                .buildAndExpand(orderId).toUri();
        PaymentResponse response = new PaymentResponse(
                payment.getOrder().getId(),
                payment.getCreditCardNumber()
        );
        return ResponseEntity.created(location).body(response);
    }

    @GetMapping("/order/{id}/receipt")
    public ResponseEntity<Receipt> getReceipt(@PathVariable("id") Long orderId) {
        Receipt receipt = orderService.getReceipt(orderId);
        return ResponseEntity.ok().body(receipt);
    }

    @ExceptionHandler
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    @ResponseBody
    public String handleOrderAlreadyPaid(OrderAlreadyPaid orderAlreadyPaid) {
        return orderAlreadyPaid.getMessage();
    }
}
```

If we were to write a unit test for this controller, we could call the `pay()`, `getReceipt()`, and `handleOrderAlreadyPaid()` methods directly with some arguments. We could then make sure that the business logic in `OrderService` is called correctly and the controller returns the responses we expect.

All good? Or is it? Let's take a closer look.

- We have annotated the endpoints with `@PostMapping` and `@GetMapping`. If we write a unit test, these annotations are not processed. How do we know that an HTTP request gets mapped to correct endpoints or that path variables get mapped with `@PathVariable`?

- What about `@RequestBody` and `@ResponseBody`? A unit test won't process these annotations either. If someone makes a POST request with a JSON body, how do we know the input is deserialized correctly?

- Once the input has been deserialized, we are validating some of the fields by annotating the request body with `@Valid`. How do we know that the validation takes place?

- As soon as the controller has called some business logic, it will return some responses to the caller. How do we know that our output is serialized correctly to JSON?

- Finally, there is an exception handler that handles exceptions thrown from the application and translates those to an HTTP status code. We could test the method itself, but how do we know that the method gets called when an exception is thrown?

As we can see, there are at least five distinct cases that unit tests are unable to verify:

1. HTTP request mapping
2. Deserialization
3. Input field validation
4. Serialization
5. Error handling

In a Spring application, the framework handles the concerns mentioned above. If we run the application, Spring will introduce all the required beans in the application context.

We need to be able to test these cases as well. When we introduce Spring to our tests, it means that we are going to write integration tests.

## Write an Integration Test With @WebMvcTest

Spring Boot offers several annotations for testing different parts of the application. These annotations scan only a specific set of auto-configuration classes that only provide what is needed to test that part of the application.

We start by testing the web layer of our application, where we only care about the previously mentioned concerns. We don't want to involve database calls in those tests, for example. 

To test our Spring MVC controllers, we can use the `@WebMvcTest` annotation. The annotation scans only beans for `@Controller`, `@ControllerAdvice`, and a few others related to the web layer.

Let's take a look at an example:

```java
@WebMvcTest(OrderController.class)
class OrderControllerTests {
    @MockBean
    private OrderService orderService;
    @Autowired
    private MockMvc mockMvc;

    @Test
    void payOrder() throws Exception {
        // ...
    }
}
```

We have annotated the test with `@WebMvcTest` and limited it to a single controller. Since we are not interested in testing the other parts of the application, we have also mocked the `OrderService` dependency with the `@MockBean` annotation.

Remember that `@WebMvcTest` does not scan beans for our services. We have to provide a bean for anything that the controller depends on. If we don't pass the controller as a parameter to `@WebMvcTest`, Spring will scan all the controllers, and we have to mock away all beans any controller depends on.

Spring Boot also autoconfigures a `MockMvc` bean for us so that we can autowire that. Using `MockMvc` fakes HTTP requests for us, making it possible to run the controller tests without starting an entier HTTP server.

Let's look at testing the different concerns mentioned before that unit tests were unable to cover.

## Verify HTTP Request Mapping And Deserialization

To verify that the controller handles HTTP requests, we call the `mockMvc.perform()` to initiate a mock HTTP request. The mock requests are constructed using builders for different HTTP methods like `post()`, `get()`, `put()` and `delete()`. Taking it further, these builders take arguments like `contentType()` and `content()`.

```java
@Test
void payOrder() throws Exception {
    Order order = new Order(1L, LocalDateTime.now(), 100.0, false);
    Payment payment = new Payment(1000L, order, "4532756279624064");

    when(orderService.pay(any(), any())).thenReturn(payment);

    mockMvc.perform(post("/order/{id}/payment", 1L)
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"creditCardNumber\": \"4532756279624064\"}"))
            .andExpect(status().isCreated());
}
```

Once we perform the request, `MockMvc` allows us to set some expectations through the `andExpect()` method. Now we can verify that the request was completed successfully by checking the HTTP status code with the `status().isCreated()` result matcher. By making the request and setting expectations, we verify that the controller responds to a specific URL.

The test also verifies that the content type is correct, and Spring correctly deserializes JSON input into Java objects annotated with `@RequestBody`. The test verifies that the controller maps any path parameters annotated with `@PathVariable`. If we had any query parameters annotated with `@RequestParam`, the test could also verify that those are mapped correctly.

## Verify Field Validation

In our controller, we have annotated the request body parameter with the `@Valid` annotation. We have also given the request object a constraint:

```java
public class PaymentRequest {
    @NotNull
    @CreditCardNumber
    private String creditCardNumber;
}
```

To verify that the fields gets validated correctly, we can provide a request that is missing the field:

```java
@Test
void paymentFailsWhenCreditCardNumberNotGiven() throws Exception {
    mockMvc.perform(post("/order/{id}/payment", 1L)
            .contentType(MediaType.APPLICATION_JSON)
            .content("{}"))
            .andExpect(status().isBadRequest());
}
```

If the validation fails, we should get an HTTP status 400 Bad Request as a result.

If the request body has more fields, it can be tempting to validate all those fields in the controller test. However, in controller tests, one could argue that it’s more important to test that validation happens. We could, for example, forget to annotate the request body parameter with the `@Valid` annotation.

It's also possible to write separate unit tests for the validation rules. We need to call the Java `Validator` methods directly and pass the validated object as an argument:

```java
class PaymentRequestTests {
    private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

    @Test
    void creditCardNumberMustNotBeNull() {
        String creditCardNumberWithInvalidChecksum = "4532756279624063";
        PaymentRequest request = new PaymentRequest(creditCardNumberWithInvalidChecksum);

        Set<ConstraintViolation<PaymentRequest>> violations = validator.validate(request);

        assertThat(violations).isNotEmpty();
    }
}
```

If we had many validation rules, we could validate the rules using such a unit test. In the controller test, we don't have to check all the rules - it is enough to trigger the validation once to make sure it happens.

## Verify Result Serialization

So far we have focused on verifying the requests but what about the responses? It is not necessarily immediately evident but Spring will automatically serialize our responses to JSON. We can verify the results of that serialization using the `jsonPath()` matcher:

```java
@Test
void getReceiptForOrder() throws Exception {
    Receipt receipt = new Receipt(
        LocalDateTime.now(),
        "4532756279624064",
        100.0);

    when(orderService.getReceipt(eq(1L))).thenReturn(receipt);

    mockMvc.perform(get("/order/{id}/receipt", 1L))
        .andExpect(jsonPath("$.date").isNotEmpty())
        .andExpect(jsonPath("$.creditCardNumber").value("4532756279624064"))
        .andExpect(jsonPath("$.amount").value(100.0));
}
```

Here we check that each of the fields for `date`, `creditCardNumber`, and `amount` have been serialized correctly in the JSON response. When we use `jsonPath()`, it takes the response body and allows us to write JSONPath expressions to validate the results.

Let's sidestep just for a moment into JSONPath expressions. Let's say we have JSON like this:

```json
{
  "books": [{
      "title": "A Song of Ice and Fire"
  }]
}
```

When we write a JSONPath expression, `$` is the root element, `.` is used as a child operator, and `[]` are used to access elements in an array. So in the above example, `$.books[0].title` means the title field of the first element in the books collection.

That is our 15 seconds introduction to JSONPath. :smile: We can also write more complex expressions, but now we know the basics that cover most of the use cases when writing tests.

## Verify Error Handling

Spring handles many error cases for us by returning some default HTTP status codes from the controller. However, if we throw any exceptions that we don't handle, we will get HTTP status 500 Internal Server Error.

Usually we want to translate these exception to more meaningful HTTP status codes. To translate our custom exception we have this kind of exception handler in our controller:

```java
@ExceptionHandler
@ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
@ResponseBody
public String handleOrderAlreadyPaid(OrderAlreadyPaid orderAlreadyPaid) {
    return orderAlreadyPaid.getMessage();
}
```

The simplest way to test that the exception handler is doing its job is to add an expectation about the HTTP status:

```java
@Test
void cannotPayAlreadyPaidOrder() throws Exception {
    when(orderService.pay(eq(1L), any())).thenThrow(OrderAlreadyPaid.class);

    mockMvc.perform(post("/order/{id}/payment", 1L)
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"creditCardNumber\": \"4532756279624064\"}"))
            .andExpect(status().isMethodNotAllowed());
}
```

To trigger the behavior, we are using Mockito to throw an `OrderAlreadyPaid` exception when the service method gets called with the given order identifier.

## Don't Forget the Business Logic

In the beginning, we established that a unit test could not handle all the responsibilities the controller has. However, we haven't tested that the business logic gets correctly called!

If we want to be sure that the controller and the service work correctly together, we have to test that the service methods are called with correct arguments. Let's take a look at a previous example:

```java
@Test
void payOrder() throws Exception {
    Order order = new Order(1L, LocalDateTime.now(), 100.0, false);
    Payment payment = new Payment(1000L, order, "4532756279624064");

    when(orderService.pay(any(), any())).thenReturn(payment);

    mockMvc.perform(post("/order/{id}/payment", 1L)
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"creditCardNumber\": \"4532756279624064\"}"))
            .andExpect(status().isCreated());
}
```

Here we return a `Payment` response if the `pay()` method of the service is called with any arguments. We don't know if the controller calls the method with the correct arguments.

We can fix the issue with a small change:

```java
    when(orderService.pay(eq(1L), eq("4532756279624064"))).thenReturn(payment);
```

Now the mock won't return anything unless the controller is calling the method with these exact arguments. Depending on other use cases, we might have to validate this differently, but this is now sufficient.

## Summary

Spring controllers have a lot of responsibilities. To test the controllers thoroughly, we have to pay attention to all these responsibilities.

Unit testing the controllers won't cover all the responsibilities the controllers have. Spring Boot provides everything we need for integration testing the controllers using `@WebMvcTest`.

In the following article of this mini-series, we will discuss integration testing our persistence layer of the application.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-webmvctest).

If you are interested in a complete course on the topic, check out [Testing Spring Boot Applications Masterclass](https://transactions.sendowl.com/stores/13745/226726) by Philip Riecks. It's the course about Spring Boot testing I would have created had I been inclined (so I have no problem recommending it to you using my affiliate link).
