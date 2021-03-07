---
title: Test Readability
date: 2021-03-06
author: Arho Huttunen
summary: 
categories:
  - Testing
tags:
  - clean code
---

In this article, we will learn different ways to improve test readability. Being able to understand what a test does, increases the maintainability of the test.

Tests document how the system should behave. They also act as self-verifying executable specifications. Sometimes we see these two things contradictory because of the level of details needed for different purposes.

In a [previous article](/dry-damp-tests), we talked about how to remove duplication while at the same time making the code more descriptive. This article is a more practical guide concentrating on test readability and expressiveness.

{{% toc %}}

## :walking: Describe Behavior With Test Name

Naming is one of the most challenging things in programming. In tests, the name of the test should describe what we test. It should also tell what kind of behavior we expect.

It is still quite a common approach to name tests after the method it's testing.

```java
@Test
void testDeposit() {
    BankAccount account = new BankAccount(100);
    account.deposit(100);
    assertEquals(200, account.getBalance());
}

@Test
void testWithdraw() {
    BankAccount account = new BankAccount(200);
    account.withdraw(50);
    assertEquals(150, account.getBalance());
}

@Test
void testWithdrawFailure() {
    BankAccount account = new BankAccount(50);
    assertThrows(InsufficientFundsException.class, () -> account.withdraw(100));
}
```

When we name tests like this, we are **stating the obvious** with the test name. We are duplicating the information we could get just by looking at what we are testing.

We don't care if we have to call a `deposit()` or a `withdraw()` method. What we need to know is what happens in different situations.

We can do better if we write the **test names in terms of behavior**. Each test name reads like a sentence, with the tested class as the subject. We can also use the `@DisplayName` annotation in JUnit 5 to create human-readable display names.

```java
@Test
@DisplayName("increases balance when a deposit is made")
void increaseBalanceWhenDepositIsMade() {
    BankAccount account = new BankAccount(100);
    account.deposit(100);
    assertEquals(200, account.getBalance());
}

@Test
@DisplayName("decreases balance when a withdrawal is made")
void decreaseBalanceWhenWithdrawalIsMade() {
    BankAccount account = new BankAccount(200);
    account.withdraw(50);
    assertEquals(150, account.getBalance());
}

@Test
@DisplayName("throws an exception when a withdrawal is made that exceeds balance")
void throwAnExceptionWhenWithdrawalIsMadeExceedingBalance() {
    BankAccount account = new BankAccount(50);
    assertThrows(InsufficientFundsException.class, () -> account.withdraw(100));
}
```

The point of writing names like this is to emphasize what the tested object _does_, not what it _is_. It would not be enough to say that we increase balance, decrease balance, or throw an exception. Notice how we are also describing why.

## :bricks: Use Structure For Readability

Each test can be structured in parts to make what we are testing obvious. One common way to do this is the _Arrange, Act, Assert_ pattern and its BDD variant _Given, When, Then_.

1. _Arrange:_ prepare any data or context required for the test
2. _Act:_ execute the target code, triggering the tested behavior
3. _Assert:_ check expectations about the behavior

When we use a standard form in our tests, they are **easier to understand**. We can quickly **find the expectations** and how it's **related to the behavior** that we want to test.

Let's take a look at an example where there is no structure.

```java
@Test
void purchaseSucceedsWhenEnoughInventory() {
    Product paperclip = new Product(1L, "Paperclip");
    Store store = new Store();
    store.addInventory(paperclip, 100);
    Customer customer = new Customer();
    assertTrue(customer.purchase(store, paperclip, 10));
    assertEquals(90, store.getInventory(paperclip));
}
```

There is not that much code, but it’s already becoming hard to track which part is doing the setup, triggering the behavior, and checking the expectations. The code is not skimmable anymore.

Now, what happens when we add a temporary variable and a couple of line breaks?

```java
@Test
void purchaseSucceedsWhenEnoughInventory() {
    Product paperclip = new Product(1L, "Paperclip");
    Store store = new Store();
    store.addInventory(paperclip, 100);
    Customer customer = new Customer();

    boolean purchaseSucceeded = customer.purchase(store, paperclip, 10);

    assertTrue(purchaseSucceeded);
    assertEquals(90, store.getInventory(paperclip));
}
```

The change is not significant, but the result is immediately much more readable. It is quite clear what part is doing the setup, what is triggering the behavior, and verifying the expectations.

### :thinking: When Common Sense Outweigh Rules

Both _Arrange, Act, Assert_ and _Given, When, Then_ are great for setting the stage. However, sometimes it can be hard to justify the presence of such a ceremony.

Let's take a look at an example.

```java
@Test
void returnFullNameOfUser() {
    Person person = aPerson().withFirstName("John").withLastName("Doe").build();

    String fullName = person.getFullName();

    assertEquals("John Doe", fullName);
}
```

Here we have added a variable so that we can separate our _Arrange_ and _Act_ steps. Is such a ceremony essential? We could go for something much more straightforward.

```java
@Test
void returnFullNameOfUser() {
    Person person = aPerson().withFirstName("John").withLastName("Doe").build();

    assertEquals("John Doe", person.getFullName());
}
```

It can help to follow the patterns with longer pieces of code, but it's somewhat unnecessary in simple cases.

## :writing_hand: Provide Just Enough Information

Tests can have either **too much or too little information**. Both cases affect how well we understand what behavior a test verifies.

One cause for too much information is that we just put all the details inline in the test. When there is too much information, **it is hard to understand what is relevant to the test**.

On the other hand, too little information makes the test obscure because we **cannot see the cause-and-effect relationship**. Having too little information is usually the result of attempting to remove duplication in the code.

### :heavy_check_mark: Only Verify What You Need

Sometimes we see people adding a lot of test conditions into a single test case. We may have attempted to reduce setup overhead or added "just one more little thing" in the test.

```java
@Test
void purchaseProducts() {
    // Arrange
    Product paperclip = new Product("Paperclip");
    Product printerPaper = new Product("Printer paper");
    Store store = new Store();
    store.addInventory(paperclip, 100);
    store.addInventory(printerPaper, 50);
    Customer customer = new Customer();
    // Act
    boolean purchaseSucceeded = customer.purchase(store, paperclip, 10);
    // Assert
    assertTrue(purchaseSucceeded);
    assertEquals(90, store.getInventory(paperclip));
    // Act
    purchaseSucceeded = customer.purchase(store, printerPaper, 100);
    // Assert
    assertFalse(purchaseSucceeded);
    assertEquals(50, store.getInventory(printerPaper));
}
```

The problem with such an approach is that it is **verifying too much functionality**. The code isn't very readable because it's hard to see what preconditions are related to what behavior.

Also, if the test fails, there are **many reasons for the test to fail**. It will be harder to pinpoint the cause of the error.

It's better to try to **assert only one condition per test**.

```java
@Test
void purchaseSucceedsWhenEnoughInventory() {
    Product paperclip = new Product("Paperclip");
    Store store = aStore().withInventory(paperclip, 100).build();
    Customer customer = new Customer();

    boolean purchaseSucceeded = customer.purchase(store, paperclip, 10);

    assertTrue(purchaseSucceeded);
}

@Test
void inventoryIsRemovedOnPurchase() {
    // ...
}

@Test
void purchaseFailsWhenNotEnoughInventory() {
    // ...
}

@Test
void inventoryIsNotRemovedOnFailedPurchase() {
    // ...
}
```

One condition per test will make the test **more readable**. It will also help with defect localization, as it's now easier to **pinpoint the cause of the error**.

### :see_no_evil: Hide Irrelevant Information

Sometimes, to test behavior, we have to construct objects that require specific data to be present. However, this data may be irrelevant for testing the behavior. We call this the **Irrelevant Information** test smell.

```java
void newPersonIsUnverified() {
    Person person = new Person("John", "Doe", 15);
    assertEquals(Person.Status.UNVERIFIED, person.getStatus());
}
```

The example is overly simple, but it showcases the issue. The information may be essential for constructing a person object, but it is irrelevant to our assertion. We could extract the unnecessary information to a factory method.

```java
@Test
void newPersonIsUnverified() {
    Person person = Persons.createPerson();
    assertEquals(Person.Status.UNVERIFIED, person.getStatus());
}
```

Now the essential setup is hidden in the factory method. The only relevant information to the test is that we create a new person.

### :eyes: Don't Hide Cause From Effect

So what happens if we have more tests with a similar setup? In the following example, only part of the data is relevant to the test.

```java
@Test
void personIsMinor() {
    Person person = new Person("John", "Doe", 15);
    assertTrue(person.isMinor());
}

@Test
void returnFullNameOfUser() {
    Person person = new Person("John", "Doe", 15);
    assertEquals("John Doe", person.getFullName());
}
```

The information may again be essential for the construction of the object, but the name is irrelevant to the first test, and the age is irrelevant to the second test.

We could again try to use a factory method here to see what happens.

```java
@Test
void personIsMinor() {
    Person person = Persons.createPerson();
    assertTrue(person.isMinor());
}

@Test
void returnFullNameOfUser() {
    Person person = Persons.createPerson();
    assertEquals("John Doe", person.getFullName());
}
```

However, now the setup won't have all the relevant information. It is unclear why the person is underage or why they have the name mentioned in the test. We call this the **Mystery Guest** test smell.

### :compass: Provide Essential Data, Show Relevant Data

Luckily, it is possible to both provide the essential information and keep the information relevant to the test. We can do that if we create a test data builder.

```java
@Test
void personIsMinor() {
    Person person = aPerson().withAge(15).build();
    assertTrue(person.isMinor());
}

@Test
void returnFullNameOfUser() {
    User user = aUser().withFirstName("John").withLastName("Doe").build();
    String fullName = user.fullName();
    assertEquals("John Doe", fullName);
}
```

Neither of the tests now has any irrelevant information. We have hidden the essential information for the construction of the object inside the test data builder.

Using this pattern helps with both removing duplication and keeping the data relevant to the tested behavior.

### :no_entry_sign: Don't Catch Exceptions Unnecessarily

We should not catch any exceptions in the test unless that is what we want to test. Sometimes you see the following kind of code.

```java
@Test
void unnecessaryCatching() {
    try {
        URL url = new URL("http://localhost");

        assertEquals("http", url.getProtocol());
    } catch (MalformedURLException e) {
        fail(e.getMessage());
    }
}
```

In this case, we know for sure that our URL is not malformed. Even if the code would make it possible to have a malformed URL, it would still be better to let the test method throw an exception. 

```java
@Test
void noNeedToCatch() throws MalformedURLException {
    URL url = new URL("http://localhost");

    assertEquals("http", url.getProtocol());
}
```

We were able to remove a lot of noise from the test. Now the test tells us precisely what we expect to happen and nothing else.

## :speech_balloon: Reveal Intent

Revealing intent by self-describing code makes it easier to comprehend what is going on in the code.

Unfortunately, it's often much neglected in test code. However, test code quality should be at least on the same level as production code!

Let's take a look at few ways to do this in the tests.

### :abcd: Use Self-Describing Names and Values

Let's take a look at a simple example. Here we are first persisting some objects, making an HTTP request, and verifying that we got the correct results.

```java
@Test
void returnVerifiedPeople() throws Exception {
    Person person1 = aPerson().withFirstName("John").build();
    Person person2 = aPerson().withFirstName("Jane").withStatus(Status.VERIFIED).build();
    personRepository.save(person1);
    personRepository.save(person2);

    client.perform(get("/person?status=VERIFIED"))
            .andExpect(jsonPath("$.[0].firstName", is("Jane")))
            .andExpect(jsonPath("$.*", hasSize(1)));
}
```

Even though the example is quite simple, there is quite many things going on.

- We have two people, but why are they called _John_ and _Jane_?
- We persist these two people
- We make an HTTP request with a query
- We make some relatively obscure looking verification

I mean, sure, it's a short test, and it doesn't look horrible. However, tests like this are not skimmable.

Let's make some improvements and discuss them.

```java
@Test
void returnVerifiedPeople() throws Exception {
    Person unverifiedPerson = aPerson().withFirstName("Unverified Person").build();
    Person verifiedPerson = aPerson().withFirstName("Verified Person").whoIsVerified().build();
    personRepository.save(unverifiedPerson);
    personRepository.save(verifiedPerson);

    client.perform(get("/person?status=VERIFIED"))
            .andExpect(jsonPath("$.[0].firstName", is("Verified Person")))
            .andExpect(jsonPath("$.*", hasSize(1)));
}
```

We have now replaced a few things with intent-revealing naming:

1. The persons are now called _Unverified Person_ and _Verified Person_. We **emphasize intent with value**. The verification now checks for _Verified Person_ instead of just _Jane_.
2. The person variables are not anymore `person1` and `person2`. We **emphasize intent with a variable name**.
3. We have added a `whoIsVerified()` method to the test data builder, which hides unnecessary details. We **emphasize intent with a method name**.

The test looks better, but it's nothing spectacular yet.

### :man_shrugging: Use Test Helper Methods

There is still a lot of details about how to perform the behavior in the test. If we are testing that when we request people with a verified status, do we care how we make the HTTP request?

Let's try to use some helper methods to hide even more details. We begin by extracting the persistence of the objects to a method.

```java
@Test
void returnVerifiedPeople() throws Exception {
    havingPersisted(aPerson().withFirstName("Unverified Person"));
    havingPersisted(aPerson().withFirstName("Verified Person").whoIsVerified());

    client.perform(get("/person?status=VERIFIED"))
            .andExpect(jsonPath("$.[0].firstName", is("Verified Person")))
            .andExpect(jsonPath("$.*", hasSize(1)));
}

private void havingPersisted(PersonBuilder personBuilder) {
    personRepository.save(personBuilder.build());
}

```

We have called the method `havingPersisted()`. We pass a builder as an argument not to have to call `build()` inside the test.

We could have called this method something like `insertIntoDatabase()`, but why didn't we do so? Well, persisting the people is not the behavior we are testing. The persistence part is a **precondition for the checked behavior**.

Next, let's extract making the request into a method.

```java
@Test
void returnVerifiedPeople() throws Exception {
    havingPersisted(aPerson().withFirstName("Unverified Person"));
    havingPersisted(aPerson().withFirstName("Verified Person").whoIsVerified());

    List<Person> people = requestVerifiedPeople();

    assertThat(people).extracting("firstName").containsOnly("Verified Person");
}

private List<Person> requestVerifiedPeople() throws Exception {
    String json = client.perform(get("/person?status=VERIFIED"))
            .andReturn().getResponse().getContentAsString();
    return toDto(json);
}
```

We have named the method `requestVerifiedPeople()`. We are not interested in _how_ we make the request; we are only interested in _what_ **behavior we trigger**.

We have implemented the method to parse the returned JSON and return a list of people. We don't have to deal with JsonPath matching, and we can use a much more fluent AssertJ assertion for the verification.

### :x: Explain Failure With Assertion Messages

Assertions have an intent too. When the test fails, we are supposed to know what went wrong. Let's take a look at a previous example we had.

```java
@Test
void increasesBalanceWhenDepositIsMade() {
    BankAccount account = new BankAccount(100);
    account.deposit(100);
    assertEquals(200, account.getBalance());
}
```

If the deposit calculation had a bug, our test would fail with the following assertion message.

```shell
expected: <200> but was: <100>
Expected :200
Actual   :100
```

Not bad, but it's not telling what exactly went wrong. When we run a single test, it's not so bad. However, if we run a bunch of tests, it's becoming worse.

Assertion frameworks allow us to provide an assertion message to make the failure more obvious. It is a feature that people are not using nearly as often as it should!

```java
@Test
void increasesBalanceWhenDepositIsMade() {
    BankAccount account = new BankAccount(100);
    account.deposit(100);
    assertEquals(200, account.getBalance(), "Account balance after deposit");
}
```

Now we can see a much more descriptive assertion message.

```shell
Account balance after deposit ==> expected: <200> but was: <100>
Expected :200
Actual   :100
```

It's much more apparent what the expected value of 200 and the actual value of 100 are. The assertion message provides context.

## :white_check_mark: Summary

Test readability has a significant impact on the maintainability of the tests. There are a few good practices that make tests easier to read.

We should always **test behavior and not the implementation**. Naming tests accordingly document the behavior.

We should **use standard structure** to find the behavior and the expectations quickly. 

Tests should **provide just enough information** to understand them. Too much or too little information can make the test obscure.

Revealing intent by self-describing names makes the code more understandable. It's good to **focus on what behavior we test, not how we test it**.

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/write-better-tests/tree/main/test-readability).

{{% callout note %}}
**Additional reading:**

:pencil2: [DRY and DAMP in Tests](/dry-damp-test/)

:pencil2: [How to Create a Test Data Builder](/test-data-builders/)

:book: [xUnit Test Patterns: Refactoring Test Code](https://amzn.to/30fANr0) by Gerard Meszaros

:book: [Growing Object-Oriented Software, Guided by Tests](https://amzn.to/2O0hHTm) by Steve Freeman, Nat Pryce
{{% /callout %}}
