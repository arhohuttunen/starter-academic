---
title: Spring Boot Unit Testing
date: 2021-04-05
author: Arho Huttunen
summary: Learn how to write unit tests for Spring Boot applications. Learn the technical details necessary to write good unit tests.
categories:
  - Testing
tags:
  - Clean code
  - JUnit 5
  - Spring
image:
  focal_point: center
  preview_only: true
---

{{< youtube Ae5ukd136pc >}}
<br/>

In this article, we will learn how to write unit tests for our Spring Boot applications. Most importantly, we will look at the technical details necessary to write good unit tests.

This article is the first one of the Spring Boot Testing mini-series. In this article, we only discuss unit testing. We will discuss integration testing in the upcoming articles.

## The Spring Boot Testing Mini-Series

1. Spring Boot Unit Testing
2. [Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest/)
3. [Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest/)
4. [Testing Serialization With Spring Boot @JsonTest](/spring-boot-jsontest/)
5. Testing REST Calls With WebTestClient And MockWebServer
6. Spring Boot Integration Testing with @SpringBootTest 

## What Is a Unit Test?

Before we start, let's first define what we mean by unit testing. Unfortunately, there is quite a bit of confusion about the size of a unit.

First, let's take a look at the definition of unit testing in Wikipedia:

>  In object-oriented programming, a unit is often an entire interface, such as a class, but could be an individual method.

Ok, so a unit could be hidden behind an interface, or it could be as small as a method. There is an important characteristic hidden here: **we should not test the implementation but the behaviour** that is exposed by the public interface.

Next, let's take a look at a definition by Michael Feathers in 2005:

A test is not a unit test if:
- It talks to the database
- It communicates across the network
- It touches the file system
- It can't run at the same time as any of your other unit tests
- You have to do special things to your environment (such as editing config files) to run it

If your test does any of the above, it's an integration test. Some people think that integration testing means that you test the entire application, but that's not true. You could, for example, integration test your data access layer in isolation.

Some people have started using the term **microtest** to describe what unit testing was supposed to be. They introduced a new term because people abuse the term **unit test** so much. 

Now that we have set that straight let's talk about unit testing in Spring applications.

## Don't Use Spring to Write Unit Tests

Wait a minute, weren't we supposed to look at unit testing with Spring Boot? Indeed, but let's take a look at what a typical Spring Boot test looks like.

Here is a service that uses field-based dependency injection:

```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private PaymentRepository paymentRepository;

    public Payment pay(Long orderId, String creditCardNumber) {
        Order order = orderRepository.findById(orderId).orElseThrow(EntityNotFoundException::new);

        if (order.isPaid()) {
            throw new PaymentException();
        }

        orderRepository.save(order.markPaid());
        return paymentRepository.save(new Payment(order, creditCardNumber));
    }
}
```

Furthermore, here is a test that tests the service:

```java
@SpringBootTest
class OrderServiceTests {
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private OrderService orderService;

    @Test
    void payOrder() {
        Order order = new Order(1L, false);
        orderRepository.save(order);

        Payment payment = orderService.pay(1L, "4532756279624064");

        assertThat(payment.getOrder().isPaid()).isTrue();
        assertThat(payment.getCreditCardNumber()).isEqualTo("4532756279624064");
    }
}
```

So, what's wrong with a test like this? Well, this is not a unit test. When we use the `@SpringBootTest` annotation, Spring loads up an application context for the test. In practice, we have started the whole application only to autowire the `OrderService` into the test.

Another problem is that we have to write orders to and read them from the database. While this could be something that we want to do in the integration tests, it's not desirable in unit tests. Remember that we want to **test the unit in isolation**.

If we want to isolate the test from the database and we are already familiar with Spring Boot and Mockito, we might ask: why not just annotate the repositories with `@MockBean` then?

```java
@SpringBootTest
class OrderServiceTests {
    @MockBean
    private OrderRepository orderRepository;
    @MockBean
    private PaymentRepository paymentRepository;
    @Autowired
    private OrderService orderService;

    @Test
    void payOrder() {
        Order order = new Order(1L, false);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentRepository.save(any())).then(returnsFirstArg());

        Payment payment = orderService.pay(1L, "4532756279624064");

        assertThat(payment.getOrder().isPaid()).isTrue();
        assertThat(payment.getCreditCardNumber()).isEqualTo("4532756279624064");
    }
}
```

We could use mocks here, and it's something we can use in our integration tests. However, it's still going to be much slower than writing a plain unit test. 

Furthermore, every time we use `@MockBean` in our tests, Spring will create a new application context in the tests, **unable to use a cached version of the context**. Having to create new context adds to the overall execution time of the tests.

Here is a quote from Spring framework documentation about unit testing:

> True unit tests typically run extremely quickly, as there is no runtime infrastructure to set up. Emphasizing true unit tests as part of your development methodology can boost your productivity.

It takes about 5 seconds to run this locally. Five seconds might not sound much, but unit tests are supposed to run in milliseconds. The execution time is not so bad with a small application, but the time goes up as your application grows.

Ok, so if we cannot use `@SpringBootTest`, what should we do then? Let's take a look.

## Make the Service Unit-Testable

Here is another quote from Spring framework documentation about unit testing:

> Dependency injection should make your code less dependent on the container than it would be with traditional Java EE development. The POJOs that make up your application should be testable in JUnit or TestNG tests, with objects instantiated by using the `new` operator, without Spring or any other container.

In the previous example, we had a service where we injected the repositories as fields. There's no way to pass the repository instances to the service if we instantiate with the `new` operator.

**The solution is not to use field injection at all**. Instead, we should use constructor injection:

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;

    public OrderService(OrderRepository orderRepository, PaymentRepository paymentRepository) {
        this.orderRepository = orderRepository;
        this.paymentRepository = paymentRepository;
    }
    
    // ...
}
```

When we provide a constructor with the repositories as parameters, Spring will automatically inject those into the service. We can also make the repository fields `final` because there's no need for them to change.

We can also reduce boilerplate code by using Lombok:

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;
    
    // ...
}
```

When the class has `final` fields, using the Lombok `@RequiredArgsConstructor` will automatically create a constructor with those parameters.

## Write a Unit Test

It's now possible to pass the repository instances to the service as constructor arguments. Now we can write a unit test for the service:

```java
class OrderServiceTests {
    private OrderRepository orderRepository;
    private PaymentRepository paymentRepository;
    private OrderService orderService;

    @BeforeEach
    void setupService() {
        orderRepository = mock(OrderRepository.class);
        paymentRepository = mock(PaymentRepository.class);
        orderService = new OrderService(orderRepository, paymentRepository);
    }
    
    @Test
    void payOrder() {
        Order order = new Order(1L, false);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentRepository.save(any())).then(returnsFirstArg());

        Payment payment = orderService.pay(1L, "4532756279624064");

        assertThat(payment.getOrder().isPaid()).isTrue();
        assertThat(payment.getCreditCardNumber()).isEqualTo("4532756279624064");
    }
}
```

Since we don't want to touch the database, we are using Mockito to replace the actual implementations of the repositories with mocks. The test now runs in milliseconds instead of seconds.

We can further reduce boilerplate in the test code if we use the `MockitoExtension` extension:

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTests {
    @Mock
    private OrderRepository orderRepository;
    @Mock
    private PaymentRepository paymentRepository;
    @InjectMocks
    private OrderService orderService;
    
    // ...
}
```

With quite a simple change, we managed to make the test independent of Spring. The test is now fast and isolated. 

{{% callout note %}}
Additional reading:

:pencil2: [Using Mockito With JUnit 5](/junit-5-mockito/)
{{% /callout %}}

## Summary

Using `@SpringBootTest` for writing plain unit tests can be considered harmful because they run slow. It is pretty easy to make our components unit-testable when we use constructor injection instead of field injection.

In addition to unit testing, we should also write integration tests. In the following article of this mini-series, we will discuss integration testing our web layer of the application.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-unit-testing).

