---
title: Test Readability
date: 2021-03-06
author: Arho Huttunen
summary: 
categories:
  - Testing
tags:
  - clean code
draft: true
---

In a [previous article](/dry-damp-tests), we talked about how to remove duplication while at the same time making the code more descriptive. This article is a more practical guide concentrating on test readability and expressiveness.

## :walking: Test Names Should Describe Behaviour

Naming is one of the most difficult things in programming. In tests, the name of the test should describe what is being tested and what kind of behaviour is expected.

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

We don't really care if we have to call a `deposit()` or a `withdraw()` method. What we really need to know is what happens in different situations.

We can do better if we write the **test names in terms of behaviour**. Each test name reads like a sentence, with the tested class as the subject. We can also use the `@DisplayName` annotation in JUnit 5 to create human-readable display names.

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
2. _Act:_ execute the target code, triggering the tested behaviour
3. _Assert:_ check expectations about the behaviour

When we use a standard form in our tests, they are **easier to understand**. We can quickly **find the expectations** and how it's **related to the behaviour** that we want to test.

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

There is not that much code, but it's already becoming hard to track which part is doing setup, which is triggering the behaviour, and which is checking the expectations. The code is not skimmable anymore.

Now what happens when we add a temporary variable and a couple of line breaks?

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

The change is not big, but the result is immediately much more readable. It is quite clear what part is doing the setup, what is triggering the behaviour, and what is verifying the expectations.

### When Common Sense Outweigh Rules

Both _Arrange, Act, Assert_ and _Given, When, Then_ are great for setting the stage. However, sometimes it can be hard to justify the presence of such ceremony.

Let's take a look at an example.

```java
@Test
void returnFullNameOfUser() {
    Person person = aPerson().withFirstName("John").withLastName("Doe").build();

    String fullName = person.getFullName();

    assertEquals("John Doe", fullName);
}
```

Here we have added a variable, so that we can separate our _Arrange_ and _Act_ steps. Is such ceremony really necessary? We could go for something much simpler.

```java
@Test
void returnFullNameOfUser() {
    Person person = aPerson().withFirstName("John").withLastName("Doe").build();

    assertEquals("John Doe", person.getFullName());
}
```

With longer pieces of code it can definitely help to follow the patterns, but in simple cases like this it's quite unnecessary.

## :speech_balloon: Reveal Intent With Method and Variable Names

## :writing_hand: Provide Just Enough Information

Tests can have either **too much or too little information**. Both cases affect how well we understand what behaviour a test verifies.

One cause for too much information is that we just put all the details inline in the test. When there is too much information, **it is hard to understand what is relevant to the test**.

On the other hand, too little information makes the test obscure because we **cannot see the cause and effect relationship**. Having too little information is usually the result of attempting to remove duplication in the code.

### Verify Only What You Need

Sometimes we see people adding a lot of test conditions into a single test case. This can happen in an attempt to reduce setup overhead, or adding "just one more little thing" in the test. 

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

The problem of such approach is that it is **verifying too much functionality**. The code isn't very readable because it's hard to see what preconditions are related to what behaviour.

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

### Hide Irrelevant Data

Sometimes to be able to test a behaviour we have to construct objects that require certain data to be present. However, this data may be irrelevant for testing the behaviour. This is called the Irrelevant Data test smell.

```java
void newPersonIsUnverified() {
    Person person = new Person("John", "Doe", 15);
    assertEquals(Person.Status.UNVERIFIED, person.getStatus());
}
```

The example is overly simple, but it showcases the issue. The information may be essential for the construction of a person object, but it is irrelevant to our assertion. We could extract the unnecessary information to a factory method.

```java
@Test
void newPersonIsUnverified() {
    Person person = Persons.createPerson();
    assertEquals(Person.Status.UNVERIFIED, person.getStatus());
}
```

Now the essential setup is hidden in the factory method. The only relevant information to the test is that a new person has been created.

### Don't Hide Cause From Effect

So what happens if we have more tests with a similar setup? In the following example only part of the data is relevant to the test.

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

However, now the setup won't have all the relevant information. It is unclear why the person is underage, or why the person has the name mentioned in the test. This is called the Mystery Guest test smell.

### Provide Essential But Only Show Relevant Data

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

Neither of the tests now have any irrelevant information. The essential information for the construction of the object has been hidden inside the test data builder.

Using this pattern helps with both removing duplication and keeping the data relevant to the tested behaviour.

{{% callout note %}}
**Additional reading:**

:pencil2: [DRY and DAMP in Tests](/dry-damp-test/)

:pencil2: [How to Create a Test Data Builder](/test-data-builders/)

:book: [xUnit Test Patterns: Refactoring Test Code](https://amzn.to/30fANr0) by Gerard Meszaros

:book: [Growing Object-Oriented Software, Guided by Tests](https://amzn.to/2O0hHTm) by Steve Freeman, Nat Pryce
{{% /callout %}}

## Summary

Test readability is the sum of various activities:

- Testing behaviour, not the implementation
- Using standard structure to easily find the behaviour and the expectations 
- Naming literals and variables to describe intent
- Using patterns like the Test Data Builder to provide just enough information

You can find the example code for this article on [GitHub](https://github.com/arhohuttunen/write-better-tests/tree/main/test-readability).
