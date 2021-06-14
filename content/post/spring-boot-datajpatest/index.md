---
title: Testing the Persistence Layer With Spring Boot @DataJpaTest
date: 2021-05-02
author: Arho Huttunen
summary: Learn how to test Spring Boot repositories using @DataJpaTest. Learn what should be tested on the persistence layer and how.
categories:
  - Testing
tags:
  - JUnit 5
  - Spring Boot
image:
  focal_point: center
  preview_only: true
---

{{< youtube wRpxKDi79mk >}}
<br/>

This article is the third part of the Spring Boot Testing mini-series. In this article, we look at how to write tests for the persistence layer.

First, we will discuss what we should be testing in the persistence layer. Then, we will look at how to deal with creating the database schema and test data for the tests.

## The Spring Boot Testing Mini-Series

1. [Spring Boot Unit Testing](/spring-boot-unit-testing/)
2. [Testing Web Controllers With Spring Boot @WebMvcTest](/spring-boot-webmvctest/)
3. Testing the Persistence Layer With Spring Boot @DataJpaTest
4. [Testing Serialization With Spring Boot @JsonTest](/spring-boot-jsontest/)
5. [Testing Spring WebClient REST Calls With MockWebServer](/spring-boot-webclient-mockwebserver/)
6. [Spring Boot Integration Testing with @SpringBootTest](/spring-boot-integration-testing/)

## What To Test?

Let's consider what is needed when we add a database connection to our application.

1. We need a **database schema** that describes how to construct our database.
2. We also have some **entities** that the framework maps to the database schema.
3. The database schema has some **constraints** to ensure the consistency of data.
4. We also need **repository interfaces** that we use to make queries to the database.

Next, let's remind ourselves about one of the characteristics of unit tests: _A test is not a unit test if it talks to the database_. So it is pretty clear that involving the database in a test makes it an integration test.

However, the only way to make sure that, for example queries work is to execute them against a running database. Unit testing the persistence layer doesn't make sense.

If we test our persistence layer in isolation, we should be able to trust that it works. There should be no need to write tests on other layers that need the database. It's simple to mock the persistence layer away. We shouldn't care about the correctness of the persistence layer in those tests.

Let's look at how we can deal with the above matters.

## Write an Integration Test With @DataJpaTest

As mentioned in the previous article of this mini-series, Spring Boot offers several annotations to test different parts of the application. To test our persistence layer and repositories, we can use the `@DataJpaTest` annotation:

```java
@DataJpaTest
class PaymentRepositoryTests {
    // ...
}
```

The `@DataJpaTest` annotation auto-configures beans for JPA-related components like our repositories. It also configures beans for things like a `DataSource`, `JdbcTemplate`, or `TestEntityManager`.

Spring Boot will also configure an embedded in-memory database if it can find one in the classpath. For example, to use the H2 database, we could have runtime dependency to it in `build.gradle`:

```gradle
dependencies {
    runtimeOnly 'com.h2database:h2'
}
```

Let's look at the database schema next.

## Create a Database Schema

By default, `@DataJpaTest` configures Hibernate to create database schema automatically based on our entities:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop
```

Tests will now create the schema before executing and drop it once they finish running. It also means that we don't have to pay attention to validating the schema.

This behavior can be helpful during the development of some functionality since our entities might change a bit. However, for production quality, we almost certainly want to use some database migration tool. Two popular tools for migrations are Flyway and Liquibase.

### Use Flyway Migrations

Using Flyway in tests doesn't need any special configuration. When Spring finds Flyway from the classpath, it's auto-configured in the tests:

```gradle
dependencies {
    runtimeOnly 'org.flywaydb:flyway-core'
}
```

When Flyway is present, it overrides the Hibernate `ddl-auto` behavior and applies it's migrations. We now know at test time that the Flyway scripts work.

However, there is a drawback when using an in-memory database for our tests; the written SQL has to work in both databases. If our migrations use a PostgreSQL-specific syntax, it might not work with the H2 database.

There are three ways to fix this. The first option is to disable Flyway in tests and let Hibernate generate the schema:

```java
@TestPropertySource(properties = {
        "spring.flyway.enabled=false",
        "spring.jpa.hibernate.ddl-auto=create-drop"
})
```

The problem with this approach is that we no longer know at the test time that Flyway scripts work as expected.

The second option is to write migrations for both databases. In a lot of cases, we can share the SQL, and there are only minor differences. However, this approach means that your tests are no longer testing the same code.

The third and preferred option is to use the same database in both the application and the tests. We'll get into that in a bit.

### Use Liquibase Migrations

The most significant difference between Flyway and Liquibase is that Liquibase allows input formats like XML, JSON or YAML in addition to SQL. The different formats act as an abstraction layer over SQL.

Just like with Flyway, if Spring finds Liquibase on the classpath, it's auto-configured in the tests:

```gradle
dependencies {
    runtimeOnly 'org.liquibase:liquibase-core'
}
```

One advantage of using Liquibase is that we can use the same scripts in our tests and the application. If we need to install our software in different environments with different databases, then Liquibase could be a better match for migrations.

However, using different databases for the application and the tests means that the tests are not executing the same code. Let's see how to avoid that problem.

## Test Against a Real Database

Like we previously saw, if we use a different database engine for our tests, we will face a couple of problems:

1. Our application code may fail in production even if the tests pass.
2. We might end up writing queries for both database engines to make it run for both the application and the tests.

Luckily, with Docker, we can quite easily run anything in a container. Testcontainers makes it easy to start up a database in a container as well:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@TestPropertySource(properties = {
        "spring.datasource.url=jdbc:tc:postgresql:13.2-alpine:///payment"
})
class PaymentRepositoryTests {
    // ...
}
```

The `AutoConfigureTestDatabase.Replace.NONE` annotation property tells Spring not to replace the database with an embedded database. If we didn't do this, Spring wouldn't be using the Postgres data source and would try to autoconfigure an embedded database instead.

When we use a specialized `jdbc:tc:postgresql:13.2-alpine:///payment` JDBC URL, Testcontainers will **automatically provide a database container instance for the tests**.

The `13.2-alpine` part refers to the tag of the PostgreSQL Docker image to use. Generally speaking, the `alpine` variants are much smaller in size.

Now our test code will be executed against a PostgreSQL instance running in Docker. We have also avoided the problems related to using two different database engines.

## Validate Entity Mapping

Creating the database schema is one thing, but we should also validate that the entities correctly map to the schema. To automatically validate the mapping, we can set Hibernate `ddl-auto` to `validate`:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

Using this feature will check that our entities match the created schema on application start-up.

For example, let's look at our `Payment` entity:

```java
@Entity
public class Payment {
    @Id
    @GeneratedValue
    private Long id;
    @OneToOne
    private Order order;
    private String creditCardNumber;
}
```

Now if we had a typo in our migration SQL:

```sql
CREATE TABLE payment(
    id BIGINT NOT NULL PRIMARY KEY,
    order_id BIGINT NOT NULL UNIQUE,
    credit_card_nubmer VARCHAR(16) NOT NULL,
    CONSTRAINT fk_order FOREIGN KEY(order_id) REFERENCES orders(id)
);
```

Hibernate would throw a `SchemaManagementException` on application start-up:

```shell
Schema-validation: missing column [credit_card_number] in table [payment]
```

There is no need to test the mapping explicitly. However, we need to be aware that the validation does not cover all cases like `unique` constraints.

## Test Custom Queries

Spring Boot Data JPA offers us several ways to write queries. The simplest option is to write inferred queries where Spring generates queries from the method name. For any more complex queries, we can either write queries using JPQL or native queries with SQL.

### Test Inferred Queries

Here is a typical JPA repository interface that uses an inferred query:

```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {
    Optional<Payment> findByOrderId(Long orderId);
}
```

We inherit default methods like `save()`, `findById()`, or `findAll()` from `JpaRepository`. Testing any of these methods would be a waste because we would be testing the framework.

Some argue that we should not test the inferred queries either because the queries are auto-generated and Spring Data validates the queries at start-up.

So, for example, if we had a typo in the above method name, the application would fail to start:

```shell
No property idd found for type Order! Did you mean 'id'? Traversed path: Payment.order.
```

However, **this is only a syntactical check and does not understand semantics**. Simple queries are easy, but any longer method name is easy to get wrong. Often "too simple to test" is exactly what we should be testing.

### Test JPQL Queries

What about if we are writing our custom JPQL queries?

```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {
    @Query("SELECT p FROM Payment p JOIN p.order o ON o.date > :after")
    List<Payment> findAllAfter(@Param("after") LocalDateTime afterDate);
}
```

This time the query is not auto-generated, and the room for error is greater. Still, Spring Data will validate any problems with the named parameters:

```shell
Using named parameters for method ...
  but parameter 'Optional[afterr]' not found in annotated query 'SELECT p FROM ...'!
```

Also, when we use Hibernate as our JPA provider, Hibernate checks for the validity of the JPQL query:

```shell
org.hibernate.hql.internal.ast.QuerySyntaxException:
  unexpected token: x near line 1, column 42 [SELECT p FROM ...]
```

Like before, this is only syntactical validation and does not validate the semantics. Spring Data doesn't know what we are trying to achieve.

### Test Native SQL Queries

Both the inferred queries and JPQL queries are an abstraction over SQL, so they don't _necessarily_ need to be run against an actual database. However, if we write any native queries, they could use a database-specific SQL dialect.

Here is a native SQL query:

```java
public interface PaymentRepository extends JpaRepository<Payment, Long> {
    @Query(value = "SELECT * FROM payment WHERE credit_card_number = :ccn", nativeQuery = true)
    List<Payment> findByCreditCardNumber(@Param("ccn") String creditCardNumber);
}
```

Spring Data will verify the named parameters, but since it's a native query that can contain database-specific SQL, no one will validate the syntax. The lack of validation makes native queries a good target for testing.

## Manage Test Data

So far, we have only talked about whether to test the queries or not. However, testing the queries often requires some data to be present.

Tests should also be independent of each other and should not rely on the results of other tests. Therefore every test should start from a known database state. 

Two common approaches to put the database into a known state are to make the tests transactional or clean up the database before each test.

Spring `@DataJpaTest` is `@Transactional` by default. Transactional tests will roll back the changes after the test has been executed.

### Manually Insert Entities

The easiest way to provide test data is to manually insert entities. We can use the Spring Boot auto-configured `TestEntityManager` to do this:

```java
class PaymentRepositoryTests {
    @Autowired
    private PaymentRepository paymentRepository;
    @Autowired
    private TestEntityManager entityManager;

    @Test
    void existingPaymentCanBeFound() {
        Order order = new Order(LocalDateTime.now(), BigDecimal.valueOf(100.0), true);
        Payment payment = new Payment(order, "4532756279624064");

        Long orderId = entityManager.persist(order).getId();
        // Flush to synchronize persistence context changes to database
        entityManager.persistAndFlush(payment);
        // Clear the context so that entities are not fetched from the first-level cache
        entityManager.clear();

        Optional<Payment> savedPayment = paymentRepository.findByOrderId(orderId);

        assertThat(savedPayment).isPresent();
        assertThat(savedPayment.get().getOrder().getPaid()).isTrue();
    }
}
```

However, in real-world projects, there are more complex entities, and they have more relationships. If there's a lot of data to insert, this approach can become cumbersome and potentially show a lot of irrelevant information.

We can use something like the [test data builder](/test-data-builders/) and the object mother pattern to tackle this problem.

The main advantages of this approach are that it's refactoring-safe and it's not hiding any information needed to understand the test.

### Use Scripts To Insert Data

An alternative to manually inserting data is to use some scripts. Spring offers an `@Sql` annotation that can be used to populate the database. Let's consider we have the following SQL:

```sql
INSERT INTO orders (id, date, amount, paid) VALUES (1, current_date, 100.0, true);
INSERT INTO payment (id, order_id, credit_card_number) VALUES (1, 1, '4532756279624064');
```

We can now refer to the SQL file using the `@Sql` annotation:

```java
@Test
@Sql("/payment.sql")
void findPaymentsByCreditCard() {
    List<Payment> payments = paymentRepository.findByCreditCardNumber("4532756279624064");

    assertThat(payments).extracting("order.id").containsOnly(1L);
}
```

The script-based approaches suffer from a couple of drawbacks:

- Refactoring our database means that we have to refactor our test datasets.
- Moving the data setup into a script can [hide relevant information](/test-readability/#-provide-just-enough-information) making the test harder to understand.

There are other tools as well, like [Database Rider](https://github.com/database-rider/database-rider). This tool utilizes DBUnit, which uses datasets to put the database into a known state. Database Rider allows the datasets to be written in XML, JSON, or YAML.

Database Rider is an excellent alternative if we don't want to make our tests transactional because it also allows cleaning up the database before a test.

## Verify Constraints

Like mentioned before, Hibernate validation is not able to catch all errors. For example, we could have a unique constraint but forgot to add it to the database schema.

In our example code, payment can only be made once for an order. If we try to save another payment for an order, it should throw a `PersistenceException`:

```java
@Test
void paymentsAreUniquePerOrder() {
    Order order = new Order(LocalDateTime.now(), BigDecimal.valueOf(100.0), true);
    Payment first = new Payment(order, "4532756279624064");
    Payment second = new Payment(order, "4716327217780406");

    entityManager.persist(order);
    entityManager.persist(first);

    assertThrows(PersistenceException.class, () -> entityManager.persistAndFlush(second));
}
```

If we didn't test such a case and forgot to add the constraint to the database schema, we could potentially end up with inconsistent data in the database.

## Use Enough Test Data

We should use as minimal datasets as possible when writing tests for the persistence layer. However, one common pitfall is to use too little data to discover potential problems in the queries.

Let's say we got distracted and forgot to add a `WHERE` in our custom query:

```java
@Query(value = "SELECT * FROM payment", nativeQuery = true)
List<Payment> findByCreditCardNumber(@Param("ccn") String creditCardNumber);
```

Now if in our test we had just one payment in the database like in a previous example, the following test would pass:

```java
@Test
@Sql("/payment.sql")
void findPaymentsByCreditCard() {
    List<Payment> payments = paymentRepository.findByCreditCardNumber("4532756279624064");

    assertThat(payments).extracting("order.id").containsOnly(1L);
}
```

The test is giving us false confidence about correctness. The test dataset needs to have at least one payment that does not match the credit card number:

```sql
INSERT INTO orders (id, date, amount, paid) VALUES (1, current_date, 100.0, true);
INSERT INTO orders (id, date, amount, paid) VALUES (2, current_date, 50.0, true);

INSERT INTO payment (id, order_id, credit_card_number) VALUES (1, 1, '4532756279624064');
INSERT INTO payment (id, order_id, credit_card_number) VALUES (2, 2, '4716327217780406');
```

Now the test would fail because the query was wrong. These are the kind of things that we easily overlook in our test data.

## Summary

To test the persistence layer, we need to create a schema and insert some data. We also need to be able to reset the database to a known state.

Using the same database engine for both the application and the tests gives us benefits like making sure that the migrations and native queries work.

We should also test the constraints for more robust tests and make sure we have enough test data for any corner cases.

In the following article of this mini-series, we will discuss testing JSON serialization separately.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/spring-boot-test-examples/tree/main/spring-boot-datajpatest).

