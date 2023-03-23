---
title: Hexagonal Architecture With Spring Boot
subtitle: The Practical Guide For a Spring Boot Implementation
date: 2023-03-20
authors:
  - arhohuttunen
summary: Spring Boot implementation of Hexagonal Architecture to develop and test the application in isolation from external technologies.
categories:
  - Software Craft
  - Spring Boot
image:
  caption: Photo by [Meggyn Pomerleau](https://unsplash.com/@yungserif) on Unsplash
  filename: stock/worker-honeybees-crawling-inside-honeycomb.jpg
_build:
  render: always
  list: never
---

Hexagonal architecture has become a popular architectural pattern for separating business logic from the infrastructure. This separation allows us to delay decisions about technology or easily replace technologies. It also makes it possible to test the business logic in isolation from external systems.

In this article, we will look at how to implement hexagonal architecture in a Spring Boot application. We will separate the business logic and the infrastructure in their own modules and see how these modules can be implemented and tested in isolation.

This article is a practical hands-on tutorial and expects that the reader has basic understanding of the principles behind hexagonal architecture. This background information is available in the [Hexagonal Architecture Explained](/hexagonal-architecture) article.

## Use Cases and Business Logic

A core attribute of the hexagonal architecture is that we can implement the business logic separately from the infrastructure. This means that we can start by focusing on the business logic alone.

We are going to start with an example application that allows a customer to order coffee. We have some rules related to the process of ordering and preparing the order.

- The customer can order a coffee and choose the type of coffee, milk, size and whether it's in store or take away.
- The customer can add more items to the order before paying.
- The customer can cancel the order before paying.
- When the order is paid, no changes are allowed.
- The customer can pay the order with a credit card.
- When the order is paid, the customer can get a receipt.
- When the order is paid, the barista can start preparing the order.
- Once the barista is finished, she can mark the order ready.
- When the order is ready, the customer can take the order and it's marked taken.

One of the advantages of hexagonal architecture is that it can encourage the preferred way of writing use cases. The use cases should live on the boundary of the application, unaware of any external technologies.

We can identify two primary actors: a customer making the order and a barista preparing the order. Knowing that the ports in hexagonal architecture are a natural fit for describing use cases of the application, this leads to introducing two primary ports: `OrderingCoffee` and `PreparingCoffee`. On the other side of application we need a couple of secondary ports for storing the orders and payments.


{{< figure src="/post/hexagonal-architecture-spring-boot/coffee-shop-use-cases.svg" caption="Coffee shop use cases" >}}

The `OrderingCoffee` and `PreparingCoffee` ports need to fulfill the requirements we have related to the process of ordering and preparing coffee.

```java
public interface OrderingCoffee {  
    Order placeOrder(Order order);  
    Order readOrder(UUID orderId);  
    Order updateOrder(UUID orderId, Order order);  
    void cancelOrder(UUID orderId);  
    Payment payOrder(UUID orderId, CreditCard creditCard);  
    Receipt readReceipt(UUID orderId);  
    Order takeOrder(UUID orderId);  
}

public interface PreparingCoffee {  
    void startPreparingOrder(UUID orderId);  
    void finishPreparingOrder(UUID orderId);  
}
```

Similarly, our secondary ports could be called simply `Orders` and `Payments` and need to be able to store and fetch orders and payments.

```java
public interface Orders {  
    Order findOrderById(UUID orderId);  
    Order save(Order order);  
    void deleteById(UUID orderId);  
}

public interface Payments {  
    Payment findPaymentByOrderId(UUID orderId);  
    Payment save(Payment payment);  
}
```

We will need some entities in our domain model. A line item in an order will hold the type of coffee, milk and size of a beverage. We will also need to track the status of the order.

```java
public class Order {  
    private UUID id = UUID.randomUUID();  
    private final Location location;  
    private final List<LineItem> items;  
    private Status status = Status.PAYMENT_EXPECTED;

    // ...
}

public record LineItem(Drink drink, Milk milk, Size size, int quantity) { }

public enum Status {  
    PAYMENT_EXPECTED,  
    PAID,  
    PREPARING,  
    READY,  
    TAKEN  
}
```

There are also classes for payments, credit cards and receipts. In this example we haven't added much behavior in them so they are just Java records.

```java
public record Payment(UUID orderId, CreditCard creditCard, LocalDate paid) { }

public record CreditCard(
        String cardHolderName,
        String cardNumber,
        Month expiryMonth,
        Year expiryYear) { }

public record Receipt(BigDecimal amount, LocalDate paid) { }
```

Next we need to implement the use cases inside our application. We are going to create a `CoffeeShop` class that implements the `OrderingCoffee` primary port. This class will also be calling the secondary ports `Orders` and `Payments`.

{{< figure src="/post/hexagonal-architecture-spring-boot/ordering-coffee-use-case-with-ports.svg" caption="Ordering coffee use case" >}}

The implementation is pretty straightforward. Here is an example of paying an order that needs both secondary ports.

```java
public class CoffeeShop implements OrderingCoffee {  
    private final Orders orders;  
    private final Payments payments;

    // ...

    @Override  
    public Payment payOrder(UUID orderId, CreditCard creditCard) {  
        var order = orders.findOrderById(orderId);  
  
        orders.save(order.markPaid());  
  
        return payments.save(new Payment(orderId, creditCard, LocalDate.now()));  
    }
```

The role of the `CoffeeShop` class is to orchestrate operations on entities and repositories. It's implementing the primary ports as use cases and using the secondary ports as repositories.

In our implementation, business logic is mostly implemented in domain entities. Here is an example of marking an `Order` paid.

```java
public class Order {  
    // ...

    public Order markPaid() {
        if (status != Status.PAYMENT_EXPECTED) {
            throw new IllegalStateException("Order is already paid");
        }
        status = Status.PAID;
        return this;
    }
}
```

Now for the second use case, we are going to implement a `CoffeeMachine` service class that implements the `PreparingCoffee` primary port.

{{< figure src="/post/hexagonal-architecture-spring-boot/preparing-coffee-use-case-with-ports.svg" caption="Preparing coffee use case" >}}

This functionality could have been implemented in the same class, but we have made a design decision here to split the implementation of different use cases in their own classes.

```java
public class CoffeeMachine implements PreparingCoffee {  
    private final Orders orders;  
  
    @Override  
    public void startPreparingOrder(UUID orderId) {  
        var order = orders.findOrderById(orderId);  
  
        orders.save(order.markBeingPrepared());  
    }  

    // ...
}
```

Since we want to delay decisions about technologies, we can create stub implementations for the secondary ports we have. The easiest way to do this for repositories is to store the entities in a map.

```java
public class InMemoryOrders implements Orders {  
    private final Map<UUID, Order> entities = new HashMap<>();  
  
    @Override  
    public Order findOrderById(UUID orderId) {  
        return entities.get(orderId);  
    }  
  
    @Override  
    public Order save(Order order) {  
        entities.put(order.getId(), order);  
        return order;  
    }  
  
    @Override  
    public void deleteById(UUID orderId) {  
        entities.remove(orderId);  
    }  
}
```

This allows for implementing the business logic without having to worry about persistence details at this point. In fact, when working in an iterative manner, the first version of the application could be shipped with these in-memory stubs.

### Acceptance Tests

These in-memory stubs can also be used for testing and enable blazing fast tests. We can start by writing some acceptance tests for our use cases. When working in a BDD manner, this would be where we would start even before the implementation.

{{< figure src="/post/hexagonal-architecture-spring-boot/coffee-shop-acceptance-test.svg" caption="Acceptance testing the application in hexagonal architecture" >}}

These tests treat the application as a black box and execute use cases only through the primary ports. This approach makes these tests more resistant to refactoring since we can refactor the implementation without having to touch the tests.

Here are some examples.

```java
class AcceptanceTests {  
    private Orders orders;  
    private Payments payments;  
    private OrderingCoffee customer;  
    private PreparingCoffee barista;  
  
    @BeforeEach  
    void setup() {  
        orders = new InMemoryOrders();  
        payments = new InMemoryPayments();  
        customer = new CoffeeShop(orders, payments);  
        barista = new CoffeeMachine(orders);  
    }  

    // ...
  
    @Test  
    void customerCanPayTheOrder() {  
        var existingOrder = orders.save(anOrder());  
        var creditCard = aCreditCard();  
  
        var payment = customer.payOrder(existingOrder.getId(), creditCard);  
  
        assertThat(payment.getOrderId()).isEqualTo(existingOrder.getId());  
        assertThat(payment.getCreditCard()).isEqualTo(creditCard);  
    }  

    @Test  
    void baristaCanStartPreparingTheOrderWhenItIsPaid() {  
        var existingOrder = orders.save(aPaidOrder());  
  
        barista.startPreparingOrder(existingOrder.getId());  
  
        assertThat(existingOrder.getStatus()).isEqualTo(Status.PREPARING);  
    }  

    // ...
}
```

People often implement acceptance tests exercising the entire application through REST endpoints and going all the way to the database. Such tests are complex to write and slow. This doesn't have to be the case!

Acceptance tests ensure that we are building the right thing. They should capture the business requirements and don't have to deal with technical concerns. The goals of the end-user should not have anything to do with the choice of technologies. This is why it's completely possible to implement them with tests that are like sociable unit tests.

Writing our acceptance tests this way makes them very fast, more resistant to refactoring and focuses on the use cases of the application.

### Unit Tests

We might also decide that there is some logic that warrants testing that logic in isolation. For example, calculating the cost of the order might be such thing. Instead of testing everything through the use cases, we might end up writing more focused tests.

Here is an example of how the cost of an order might be calculated.

```java
public class Order {
    // ...
    
    public BigDecimal getCost() {
        return items.stream()
                .map(LineItem::getCost)
                .reduce(BigDecimal::add)
                .orElse(BigDecimal.ZERO);
    }
}
```

Each line item knows how to calculate its own cost. For simplicity, we assume that every small drink costs 4.0 and large drink 5.0.

```java
public record LineItem(Drink drink, int quantity, Milk milk, Size size) {  
    BigDecimal getCost() {  
        var price = BigDecimal.valueOf(4.0);  
        if (size == Size.LARGE) {  
            price = price.add(BigDecimal.ONE);  
        }  
        return price.multiply(BigDecimal.valueOf(quantity));  
    }  
}
```

To test this logic, we can choose to test the `Order` class directly with unit tests.

{{< figure src="/post/hexagonal-architecture-spring-boot/order-cost-unit-test.svg" caption="Unit testing business logic" >}}
Here is an example unit test that creates some orders and then verifies that the cost of the order is calculated correctly.

```java
public class OrderCostTest {

    private static Stream<Arguments> drinkCosts() {
        return Stream.of(
                arguments(1, Size.SMALL, BigDecimal.valueOf(4.0)),
                arguments(1, Size.LARGE, BigDecimal.valueOf(5.0)),
                arguments(2, Size.SMALL, BigDecimal.valueOf(8.0))
        );
    }

    @ParameterizedTest(name = "{0} drinks of size {1} cost {2}")
    @MethodSource("drinkCosts")  
    void orderCostIsBasedOnQuantityAndSize(
            int quantity, Size size, BigDecimal expectedCost) {

        var order = new Order(Location.TAKE_AWAY, List.of(
                new OrderItem(Drink.LATTE, quantity, Milk.WHOLE, size)
        ));

        assertThat(order.getCost()).isEqualTo(expectedCost);
    }

    @Test
    void orderCostIsSumOfLineItemCosts() {
        var order = new Order(Location.TAKE_AWAY, List.of(
                new OrderItem(Drink.LATTE, 1, Milk.SKIMMED, Size.LARGE),
                new OrderItem(Drink.ESPRESSO, 1, Milk.SOY, Size.SMALL)
        ));
        assertThat(order.getCost()).isEqualTo(BigDecimal.valueOf(9.0));
    }
}
```

Note that we have decided to call the test class `OrderCostTest` instead of just `OrderTest`. This emphasizes the idea that we should be testing behavior and not classes or methods.

The decision whether to use more focused unit tests is a case by case choice. We can avoid a combinatorial explosion in tests on a higher abstraction level by writing some more focused tests instead.

## Primary Adapters

The next logical step would be to add some primary adapters. Although the application itself is unaware of who is calling its use cases, we have to provide means for the outside world to communicate with the application.

Let's look at an example that implements a REST endpoint taking in an order. The controller implementation is as thin as possible, maps the incoming request to something understood by the application and calls a primary port of the application.

```java
@Controller  
@RequiredArgsConstructor  
public class OrderController {  
    private final OrderingCoffee orderingCoffee;  
  
    @PostMapping("/order")  
    ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request,
            UriComponentsBuilder uriComponentsBuilder) {

        var order = orderingCoffee.placeOrder(request.toDomain());  
        var location = uriComponentsBuilder.path("/order/{id}")
                .buildAndExpand(order.getId())
                .toUri();  
        return ResponseEntity.created(location).body(OrderResponse.fromDomain(order));  
    }

    // ...
}
```

Here we have put the mapping code between the domain and the response in the response object itself.

```java
public record OrderResponse(
        Location location,
        List<OrderItemResponse> items,
        BigDecimal cost,
        Status status
) {
    public static OrderResponse fromDomain(Order order) {
        return new OrderResponse(
                order.getLocation(),
                order.getItems().stream().map(OrderItemResponse::fromDomain).toList(),
                order.getCost(),
                order.getStatus()
        );
    }
}
```

Note that we have chosen to use separate `OrderRequest` and `OrderResponse` objects. This is because the write and read models don't have to be the same and could have different properties.

{{< figure src="/post/hexagonal-architecture-spring-boot/order-controller-model-mapping.svg" caption="Mapping of models in order controller" >}}

{{% callout note %}}
Strictly speaking, this kind of mapping strategy does not prevent domain logic leaking into the primary adapters. If we wanted complete isolation, we would define something like a `PlacingOrdersCommand` model to go with the `PlacingOrders` interface and deny direct access to the `Order` from adapters. This would come with the cost of even more mapping code and is not justifiable for all applications.
{{% /callout %}}

### Configuring the Application

Having the primary adapters ready is not enough. If we would now try to start the application, it would fail because of missing domain beans. Thus, we have to let Spring now how to wire our domain classes `CoffeeShop` and `CoffeeMachine` that do all the work.

The typical way to do this in a Spring Boot application would be to annotate the `CoffeeShop` and `CoffeeMachine` classes with the `@Service` annotation. However, if we want to keep the framework details out from the application, we cannot do this.

So, how should we do this? One could create some configuration classes and add bean configurations for the required service classes manually. This would however be quite tedious in a large code base.

Instead, we can create a configuration that scans classes annotated with a custom annotation. First, we are going to create our own annotation inside the application.

```java
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.TYPE)  
@Inherited  
public @interface UseCase {  
}
```

This is just a marker interface that also makes it clear that a class is implementing a use case.

```java
@UseCase  
public class CoffeeShop implements OrderingCoffee {
    // ...
}
```

Next, let's add a configuration that scans for classes annotated with the annotation and adds beans for them.

```java
@Configuration
@ComponentScan(
        basePackages = "com.arhohuttunen.coffeeshop.application",
        includeFilters = @ComponentScan.Filter(
                type = FilterType.ANNOTATION, value = UseCase.class
        )
)
public class DomainConfig {
}
```

Now we can annotate our domain classes without any framework specific annotations. Spring Boot will find any classes annotated with the custom annotation and automatically create beans for them.

### Integration Tests for Primary Adapters

To test the controller, we could mock the primary ports and then verify that the controller interacts correctly with the application. While this is a common approach and definitely would work, it has two major drawbacks.

First, it forces us to change the tests every time we refactor the interfaces between the adapters and the application. Second, testing the model mapping code is more error prone because we have to check that the ports are called with correct arguments.

We are going to use another approach for testing the primary adapters, where we inject the application as such and reuse the in-memory stubs previously created. This approach is more resistant to refactoring and exercises the mapping code as part of the flow.

{{< figure src="/post/hexagonal-architecture-spring-boot/order-controller-integration-test.svg" caption="Testing the order controller with integration tests" >}}

In our tests we need a test configuration that has bean configurations for the in-memory stubs.

```java
@TestConfiguration
@Import(DomainConfig.class)
public class DomainTestConfig {
    @Bean
    Orders orders() {
        return new InMemoryOrders();
    }
  
    @Bean
    Payments payments() {
        return new InMemoryPayments();
    }
}
```

This first imports the domain configuration and then adds required bean configurations for the stubs.

{{% callout note %}}
If this would be the first shipped iteration of our application, we'd need to have bean configurations for the in-memory stubs inside the application, not only for the tests.
{{% /callout %}}

Now our tests have to import this test configuration. We will be able to use the secondary ports to interact with in-memory stubs.

```java
@WebMvcTest  
@Import(DomainTestConfig.class)
public class OrderControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private Orders orders;

    // ...

    @Test  
    void updateOrder() throws Exception {
        var order = orders.save(anOrder());
  
        mockMvc.perform(post("/order/{id}", order.getId())
                .contentType(MediaType.APPLICATION_JSON_VALUE)
                .content(orderJson))
                .andExpect(status().isOk());
    }
}
```

There is no need to use a mocking framework or stub method calls. There is no need to verify the arguments of those method calls either. We simply insert some objects into our in-memory stub to set up the desired state. This also exercises the mapping code without having to test that separately.

With this approach we have to be careful to not start testing the application business logic in the adapter tests. Our tests should focus only on testing the controller responsibilities.

{{% callout note %}}
**Additional reading:**

[Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest)
{{% /callout %}}

## Secondary Adapters

When we have an entry point to the system ready, it's time to implement secondary adapters. We are going to need persistence of the orders.

{{< figure src="/post/hexagonal-architecture-spring-boot/orders-jpa-adapter-model-mapping.svg" caption="Mapping of models in orders JPA adapter" >}}
In the example we are using JPA because it's what a lot of people are familiar with. Here we have the dependency inversion principle in action, where the application does not depend on the adapter directly but depends on a secondary port, which is then implemented by the adapter.

```java
@Component  
@RequiredArgsConstructor  
public class OrdersJpaAdapter implements Orders {  
    private final OrderJpaRepository orderJpaRepository;  
  
    @Override  
    public Order findOrderById(UUID orderId) {  
        return orderJpaRepository.findById(orderId)  
                .map(OrderEntity::toDomain)  
                .orElseThrow();  
    }  

    // ...
}
```

The `OrdersJpaAdapter` is responsible for making the translation between the domain and the JPA entities. This means that we can create an `OrderJpaRepository` interface which is called directly from the adapter.

```java
public interface OrderJpaRepository extends JpaRepository<OrderEntity, UUID> { }
```

The `OrderEntity` itself holds the `jakarta.persistence` annotations for ORM. A lot of applications pollute the domain model with such annotations. Here we have a clean separation of those concerns with the cost of having to do mapping between the models.

```java
@Entity
public class OrderEntity {
    @Id
    private UUID id;

    @Enumerated
    @NotNull
    private Location location;
    
    @Enumerated  
    @NotNull
    private Status status;
    
    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = "order_id")
    private List<OrderItemEntity> items;

    public Order toDomain() {
        return new Order(
                id,
                location,
                items.stream().map(LineItemEntity::toDomain).toList(),
                status
        );  
    }

    // ...
}
```

Here we have also put the mapping code between the domain and the JPA entity in the `OrderEntity` class itself. This could also be extracted into its own class but we are trying to keep things simple.

### Handling Transactions

Now that we have added persistence to our system, there is one good question: where should we put the transactions?

A use case is a natural unit of work. As such it would make sense to wrap a use case into a transaction. The traditional way to do this would be to annotate a method implementing the use case with the `@Transactional` annotation. While this would be very practical, if we truly want to keep frameworks out from the application core, we can do better.

It turns out we can utilize aspect-oriented programming for this. This is not a guide to aspect-oriented programming but the idea is to add behavior to existing code without having to modify the code itself.

First, we need something that executes a piece of code inside a transaction.

```java
public class TransactionalUseCaseExecutor {
    @Transactional
    <T> T executeInTransaction(Supplier<T> execution) {
        return execution.get();
    }
}
```

Next, we will create an aspect that finds a join point for a method and executes that method with the executor we previously created. This will effectively execute that method inside a transaction.

```java
@Aspect
@RequiredArgsConstructor
public class TransactionalUseCaseAspect {

    private final TransactionalUseCaseExecutor transactionalUseCaseExecutor;

    @Pointcut("@within(useCase)")
    void inUseCase(UseCase useCase) {

    }
  
    @Around("inUseCase(useCase)")
    Object useCase(ProceedingJoinPoint proceedingJoinPoint, UseCase useCase) {
        return transactionalUseCaseExecutor.executeInTransaction(() -> {
            try {
                return proceedingJoinPoint.proceed();
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```

Basically the code finds any classes annotated with `@UseCase` and applies the `TransactionalUseCaseAspect`  to the methods in that class. This brings another useful quality to the `@UseCase` annotation we created.

Finally, we need a configuration to enable the aspect.

```java
@Configuration
@EnableAspectJAutoProxy
public class UseCaseTransactionConfiguration {
    @Bean  
    TransactionalUseCaseAspect transactionalUseCaseAspect(
            TransactionalUseCaseExecutor transactionalUseCaseExecutor
    ) {
        return new TransactionalUseCaseAspect(transactionalUseCaseExecutor);
    }

    @Bean
    TransactionalUseCaseExecutor transactionalUseCaseExecutor() {
        return new TransactionalUseCaseExecutor();
    }
}
```

Now our use cases are transactional automatically when a class is annotated with the `@UseCase` annotation. This is one example of how we can add behavior that is not central to the business logic without cluttering the code with that functionality.

### Integration Tests for Secondary Adapters



{{< figure src="/post/hexagonal-architecture-spring-boot/orders-jpa-adapter-integration-test.svg" caption="Testing secondary adapters with integration tests" >}}

Now for the tests.

```java
@DataJpaTest
@ComponentScan("com.arhohuttunen.coffeeshop.adapter.out.persistence")
public class OrdersJpaAdapterTest {
    @Autowired
    private Orders orders;

    @Autowired
    private OrderJpaRepository orderJpaRepository;

    @Test
    void creatingOrderReturnsPersistedOrder() {
        var order = new Order(Location.TAKE_AWAY, List.of(
                new OrderItem(Drink.LATTE, 1, Milk.WHOLE, Size.SMALL))
        );

        var persistedOrder = orders.save(order);

        assertThat(persistedOrder.getLocation()).isEqualTo(Location.TAKE_AWAY);
        assertThat(persistedOrder.getItems()).containsExactly(
                new OrderItem(Drink.LATTE, 1, Milk.WHOLE, Size.SMALL)
        );
    }
}
```

Since the `@DataJpaTest` annotation only scans certain beans, we have to also scan for our adapter classes.

In this example, we are using the H2 database, but in a production-ready application we'd use something else like PostgreSQL. In such case, it would be better to test the persistence adapter against the real database e.g. using Testcontainers.

{{% callout note %}}
**Additional reading:**

[Testing the Persistence Layer With Spring Boot @DataJpaTest](/spring-boot-datajpatest)
{{% /callout %}}

## End-To-End Tests

Although we should rely that most of the functionality is already tested on lower levels, there can be some gaps in the testing that those tests miss. To gain confidence that everything works correctly, we also have to implement a small number of end-to-end tests or broad integration tests.

{{< figure src="/post/hexagonal-architecture-spring-boot/coffee-shop-end-to-end-test.svg" caption="End-to-end testing the coffee shop system" >}}

Here we have opted for implementing broad integration tests with a mocked environment, but you could also write end-to-end tests with a running server.

```java
@SpringBootTest
@AutoConfigureMockMvc
class CoffeeShopApplicationTests {
    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private CoffeeMachine coffeeMachine;

    @Test
    void processNewOrder() throws Exception {
        var orderId = placeOrder();
        payOrder(orderId);
        prepareOrder(orderId);
        readReceipt(orderId);
        takeOrder(orderId);
    }

    @Test
    void cancelOrderBeforePayment() throws Exception {
        var orderId = createOrder();
        cancelOrder(orderId);
    }

    private UUID placeOrder() throws Exception {  
        var location = mockMvc.perform(post("/order")  
                        .contentType(MediaType.APPLICATION_JSON_VALUE)  
                        .content("""  
                        {
                            "location": "IN_STORE",
                            "items": [{
                                "drink": "LATTE",
                                "quantity": 1,
                                "milk": "WHOLE",
                                "size": "LARGE"
                            }]
                        }
                        """))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getHeader(HttpHeaders.LOCATION);

        return location == null ? null :
                UUID.fromString(location.substring(location.lastIndexOf("/") + 1));
    }

    // ...
}
```

We have created helper methods for each of the endpoint calls matching the steps in our use case. The tests touch each of the endpoints at least once but don't make excessive assertions. Each of the steps only make sure that the request succeeded.

These tests cover any gaps in our testing on the lower levels. Writing these tests like this makes it easier to write them and focuses only on making sure that everything is wired together correctly.

{{% callout note %}}
**Additional reading:**

[Spring Boot Integration Testing With @SpringBootTest](/spring-boot-integration-testing)
{{% /callout %}}

## Structuring the Application

In our example application, there are just two gradle modules in the `settings.gradle` file.

```groovy
include 'coffeeshop-application'
include 'coffeeshop-infrastructure'
```

The difference between these modules is that the `coffeeshop-application` holds all the business logic and use cases of the application and does not depend on Spring Boot at all. In fact, the only dependencies it has are JUnit 5 and AssertJ.

```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.9.2'
    testImplementation 'org.assertj:assertj-core:3.24.2'
}
```

This is where we can implement acceptance tests as sociable unit tests for the business logic. We can also add some more solitary unit tests for testing parts of the logic in isolation.

The second module called `coffeeshop-infrastructure` implements all the adapters and adds all the configurations needed for the Spring Boot application. A notable difference is that this module has all the Spring Boot dependencies and also adds `coffeeshop-application` as a dependency.

```groovy
dependencies {
    implementation project(':coffeeshop-application')
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // ...
}
```

This is where we test those adapters with integration tests and add a few end-to-end tests to make sure the entire solution is wired together correctly.

Internally, these modules are structured with a package structure that differentiates between primary and secondary ports and adapters.

```text
├── coffeeshop-application
|   └── application
|       ├── in
|       |    ├── OrderingCoffee.java
|       |    └── PreparingCoffee.java
|       ├── order
|       |    ├── LineItem.java
|       |    └── Order.java
|       ├── out
|       |    ├── Orders.java
|       |    └── Payments.java
|       └── payment
|            ├── CreditCard.java
|            ├── Payment.java
|            └── Receipt.java
└── coffeeshop-infrastructure
    ├── adapter
    |   ├── in
    |   |   └── rest
    |   |       ├── OrderController.java
    |   |       ├── PaymentController.java
    |   |       └── ReceiptController.java
    |   └── out
    |       └── persistence
    |           ├── OrderJpaRepository.java
    |           ├── OrdersJpaAdapter.java
    |           ├── PaymentJpaRepository.java
    |           └── PaymentsJpaAdapter.java
    ├── config
    |   └── DomainConfig.java
    └── CoffeeShopApplication.java
```

## Summary

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-hexagonal-architecture).
