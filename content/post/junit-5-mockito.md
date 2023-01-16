---
title: Using Mockito With JUnit 5
date: 2020-11-28
authors:
  - arhohuttunen
summary: Learn how to use the Mockito mocking framework with JUnit 5. Learn both the test framework independent way, and using the Mockito JUnit 5 extension.
categories:
  - Java
tags:
  - testing
featured: true
image:
  focal_point: top
  filename: edited/junit-5-mockito.png
---

In this article, we will learn how to use the Mockito mocking framework with JUnit 5. We will learn a test framework independent way, and how to use the Mockito JUnit 5 extension.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

{{< youtube p7_cTAF39A8 >}}
<br/>

## Manual Initialization

Before doing anything else, we have to add the Mockito dependency.

```gradle
dependencies {
    testImplementation('org.mockito:mockito-core:3.12.4')
}
```

If we just want to create a mock to be injected into another object, the simplest way is to call the `Mockito.mock()` method. The method takes the class of the object to be instantiated as a parameter.

```java
class MockitoManualTest {

    private OrderRepository orderRepository;
    private OrderService orderService;

    @BeforeEach
    void initService() {
        orderRepository = mock(OrderRepository.class);
        orderService = new OrderService(orderRepository);
    }

    @Test
    void createOrderSetsTheCreationDate() {
        Order order = new Order();
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
}
```

Manual initialization can be a legit solution if we don't have many mocks.

**Pros:**

- Most control on the mocks we need to create

**Cons:**

- Can become quite verbose
- Does not validate framework usage or detect incorrect stubbing

## Annotation Based Initialization

A declarative alternative to calling the `Mockito.mock()` method is to annotate a field as a mock with the `@Mock` annotation. We have to call a particular method to initialize the annotated objects.

In Mockito 2 there is a `MockitoAnnotations.initMock()` method, which is deprecated and replaced with `MockitoAnnotations.openMocks()` in Mockito 3. The `MockitoAnnotations.openMocks()` method returns an instance of `AutoClosable` which can be used to close the resource after the test.

```java
public class MockitoAnnotationTest {

    @Mock
    private OrderRepository orderRepository;
    private AutoCloseable closeable;
    private OrderService orderService;

    @BeforeEach
    void initService() {
        closeable = MockitoAnnotations.openMocks(this);
        orderService = new OrderService(orderRepository);
    }

    @AfterEach
    void closeService() throws Exception {
        closeable.close();
    }

    @Test
    void createOrderSetsTheCreationDate() {
        Order order = new Order();
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
}
```

The `MockitoAnnotations.openMocks(this)` call tells Mockito to scan this test class instance for any fields annotated with the `@Mock` annotation and initialize those fields as mocks.

**Pros:**

- Easy to create mocks
- Very readable

**Cons:**

- Does not validate framework usage or detect incorrect stubbing

## Automatic Mock Injection

We can also tell Mockito to inject mocks automatically to a field annotated with `@InjectMocks`.

When `MockitoAnnotations.openMocks()` is called, Mockito will:
 - Create mocks for fields annotated with the `@Mock` annotation
 - Create an instance of the field annotated with `@InjectMocks` and try to inject the mocks into it

Using `@InjectMocks` is the same as we did when instantiating an instance manually, but now automatic.

```java
public class MockitoInjectMocksTests {

    @Mock
    private OrderRepository orderRepository;
    private AutoCloseable closeable;
    @InjectMocks
    private OrderService orderService;

    @BeforeEach
    void initService() {
        closeable = MockitoAnnotations.openMocks(this);
    }

    @AfterEach
    void closeService() throws Exception {
        closeable.close();
    }

    @Test
    void createOrderSetsTheCreationDate() {
        Order order = new Order();
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
}
```

Mockito will first try to inject mocks by constructor injection, followed by setter injection, or field injection.

**Pros:**

- Easy to inject mocks

**Cons:**

- Doesn't enforce usage of constructor injection

{{% callout warning %}}
It is not recommended to use field or setter injection. Using constructor injection, we can be 100% sure no one instantiates the class without injecting its dependencies.
{{% /callout %}}

## Mockito JUnit 5 Extension

There is also a Mockito extension for JUnit 5 that will make the initialization even simpler.

To be able to use the extension we have to first add the dependency to it.

```gradle
dependencies {
    testImplementation('org.mockito:mockito-junit-jupiter:3.12.4')
}
```

Now we can apply the extension and get rid of the `MockitoAnnotations.openMocks()` method call.

```java
@ExtendWith(MockitoExtension.class)
public class MockitoExtensionTest {

    @Mock
    private OrderRepository orderRepository;
    private OrderService orderService;

    @BeforeEach
    void initService() {
        orderService = new OrderService(orderRepository);
    }

    @Test
    void createOrderSetsTheCreationDate() {
        Order order = new Order();
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
}
```

Note that we can use the `@InjectMocks` annotation with the `MockitoExtension` as well to simplify the setup further.

```java
@ExtendWith(MockitoExtension.class)
public class MockitoExtensionInjectMocksTest {

    @Mock
    private OrderRepository orderRepository;
    @InjectMocks
    private OrderService orderService;

    @Test
    void createOrderSetsTheCreationDate() {
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order order = new Order();

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
}
```

If we do not want to share mock variables across all test cases, we can also inject mock objects to method parameters.

```java
    @Test
    void createOrderSetsTheCreationDate(@Mock OrderRepository orderRepository) {
        OrderService orderService = new OrderService(orderRepository);
        when(orderRepository.save(any(Order.class))).then(returnsFirstArg());

        Order order = new Order();

        Order savedOrder = orderService.create(order);

        assertNotNull(savedOrder.getCreationDate());
    }
```

Injecting mocks to method parameters works both on the lifecycle methods and on the test methods themselves.

**Pros:**

- No need to call `MockitoAnnotations.openMocks()`
- Validates framework usage and detects incorrect stubbing
- Easy to create mocks
- Very readable

**Cons:**

- Need an extra dependency on `org.mockito:mockito-junit-jupiter`

## Summary

There are three different ways of using Mockito with JUnit 5. First two approaches work independently of the used framework, while the third one utilizes the Mockito JUnit 5 extension.

Mocks can be created and initialized by:

- Manually creating them by calling the  `Mockito.mock()` method
- Annotating them with the `@Mock` annotation, and initializing by calling the `MockitoAnnotations.openMocks()` method
- Annotating them with the `@Mock` annotation, and applying the `MockitoExtension` extension to the test class

The example code for this guide can be found on [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-mockito).
