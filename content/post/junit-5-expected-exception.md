---
title: "JUnit 5 Expected Exception: How to assert an exception is thrown?"
date: 2019-10-01
authors:
  - arhohuttunen
summary: Learn how to assert an exception is thrown using JUnit 5. Also, learn how to check the error message of the thrown exception.
categories:
  - Java
tags:
  - testing
image:
  filename: edited/junit-5-laptop-with-service-error.png
featured: true
---

In this article, we will learn how to assert an exception is thrown using JUnit 5. In addition, we will learn how to check the error message of the thrown exception.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

## Overview

In order to make sure our error handling works correctly, it can be useful to verify that a piece of code throws a specific exception under certain conditions.

## Asserting Thrown Exception

Asserting a piece of code throws a specific exception can be done with the `assertThrows()` method in JUnit 5:

```java
@Test
void notEnoughFunds() {
    BankAccount account = new BankAccount(9);
    assertThrows(NotEnoughFundsException.class, () -> account.withdraw(10),
            "Balance must be greater than amount of withdrawal");
}
```

In this example the implementation will throw `NotEnoughFundsException` if we try withdraw more money from a bank account that the account balance allows.

### Failing Assertions

Now, let's say in our example we forgot to check the balance before the withdrawal. If it doesn't throw an exception, the test will fail with an error message:

```bash
Balance must be greater than amount of withdrawal ==> Expected com.arhohuttunen.junit5.exception.NotEnoughFundsException to be thrown, but nothing was thrown.
org.opentest4j.AssertionFailedError: Balance must be greater than amount of withdrawal ==> Expected com.arhohuttunen.junit5.exception.NotEnoughFundsException to be thrown, but nothing was thrown.
```

Furthermore, let's say in our example we forgot to initialize the balance, and our code will throw a `NullPointerException`. If it throws an unexpected exception, the test will fail with a different error message:

```bash
Balance must be greater than amount of withdrawal ==> Unexpected exception type thrown ==> expected: <com.arhohuttunen.junit5.exception.NotEnoughFundsException> but was: <java.lang.NullPointerException>
org.opentest4j.AssertionFailedError: Balance must be greater than amount of withdrawal ==> Unexpected exception type thrown ==> expected: <com.arhohuttunen.junit5.exception.NotEnoughFundsException> but was: <java.lang.NullPointerException>
```

## Asserting Exception Message

In addition, there might be cases where we want to verify some information about the exception, such as the error message or the cause. In such cases we can capture the thrown exception:

```java
@Test
void notEnoughFundsWithMessage() {
    BankAccount account = new BankAccount(0);
    Throwable thrown = assertThrows(NotEnoughFundsException.class, () -> account.withdraw(100));
    assertEquals("Attempted to withdraw 100 with a balance of 0", thrown.getMessage());
}
```

## Summary

JUnit 5 makes it easy to assert that an expected exception is thrown with the `assertThrows()` method. Moreover, we can capture the thrown exception to check for further information like the error message.

The example code for this guide can be found on [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-expected-exception).
