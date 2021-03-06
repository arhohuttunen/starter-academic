---
title: "JUnit 5 Assertions: Verifying Test Results"
date: 2019-04-19
author: Arho Huttunen
summary: Learn how to verify test results using JUnit 5 assertions. Learn the basic assertion methods, error message customization and assertion grouping.
categories:
  - Testing
tags:
  - JUnit 5
featured: true
---

## Overview

In this article, we will learn how to verify test results using JUnit 5 assertions.
We will learn the basic methods for assertions, how to customize our error messages, and how to run multiple assertions as a group.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

## Assertions

JUnit 5 assertions make it easier to verify that the expected test results match the actual results.
If any assertion of a test will fail, the test will fail. Similarly, if all assertions of a test pass, the test will pass.

The JUnit 5 assertions are static methods in the `org.junit.jupiter.api.Assertions` class.
Let's take a closer look at the use cases of these methods.

### Values

When verifying results, one of the most common scenarios is that we want to ensure that an expected value is equal to the actual value.
JUnit 5 has `assertEquals()` and `assertNotEquals()` methods to compare the equality and inequality of values.

In this example, we have a simple `Calculator` class that adds two numbers together.
We want to make sure the result is correct:

```java
@Test
void addNumbers() {
    Calculator calculator = new Calculator();
    assertEquals(3, calculator.add(1, 2));
}
```

If the assertion failed, we would see both the expected and actual values in the error message:

```bash
org.opentest4j.AssertionFailedError:
Expected :3
Actual   :2
```

### Boolean Values

Commonly, we want to make sure if a returned value is true or false.
We could use the `assertEquals()` method but JUnit 5 has `assertTrue()` and `assertFalse()` convenience methods to do this.

In this example we make sure the first name of a person starts with a certain letter:

```java
@Test
void firstNameStartsWithJ() {
    Person person = new Person("John", "Doe");
    assertTrue(person.getFirstName().startsWith("J"));
}
```

To assert that something is not true, we would use `assertFalse()` similarly.

### Null Values

Sometimes we need to ensure that an object is null or not null.
To do this we can use the JUnit 5 assertion methods `assertNull()` and `assertNotNull()`.

In this example we are making sure that a field in the `Person` class has a non-null value:

```java
@Test
void personHasFirstName() {
    Person person = new Person("John", "Doe");
    assertNotNull(person.getFirstName());
}
```

If the assertion failed, we would see an error message:

```bash
org.opentest4j.AssertionFailedError: expected: not <null>
```

While we sometimes might need to assert `null` values, we should generally avoid passing and returning `null` values.

{{% callout note %}}
**Additional reading:**

:pencil2: [Avoiding Unnecessary Null Checks](/avoiding-unnecessary-null-checks/)
{{% /callout %}}

### Iterables

Sometimes we need to make sure a collection has the items that we expect.
We might, for example, want to verify that our sorting algorithm works.

The `assertIterableEquals()` method in JUnit 5 ensures an iterable object has the items we expect.
We can compare any classes that implement the `Iterable` interface.

In this example we are asserting a list has its items in correct order after sorting:

```java
@Test
void iterablesEqual() {
    final List<String> list = Arrays.asList("orange", "mango", "banana");
    final List<String> expected = Arrays.asList("banana", "mango", "orange");

    Collections.sort(list);

    assertIterableEquals(expected, list);
}
```

Let's say that our sorting algorithm fails, and it doesn't sort the array.
The assertion would fail with an error message:

```bash
org.opentest4j.AssertionFailedError: iterable contents differ at index [0],
Expected :<banana>
Actual   :<orange>
```

The `assertIterableEquals()` method also checks that the array lengths match.
If we were to add another element to the `array`, the assertion would fail with another error message:

```bash
org.opentest4j.AssertionFailedError: iterable lengths differ,
Expected :<3>
Actual   :<4>
```

{{% callout note %}}
Two iterables are equal if they both are `null` or empty or contain the same values.
{{% /callout %}}

### Arrays

Asserting arrays is very similar to asserting iterables.
We can use the `assertArrayEquals()` method from JUnit 5:

```java
@Test
void arraysEqual() {
    final int[] array = { 3, 2, 1 };
    final int[] expected = { 1, 2, 3 };

    Arrays.sort(array);

    assertArrayEquals(expected, array);
}
```

{{% callout note %}}
Two arrays are equal if they both are `null` or empty or contain the same values.
{{% /callout %}}

### Objects by Value

There are a couple of things we need to take into consideration when asserting that two objects are equal.

In this example, we have a `Person` class with a first and last name.
We want to assert the two instances of `Person` are equal:

```java
@Test
void personsAreSame() {
    Person john = new Person("John", "Doe");
    Person doe = new Person("John", "Doe");

    assertEquals(john, doe);
}
```

Running the test we can see that it fails with a rather cryptic error message:

```bash
org.opentest4j.AssertionFailedError:
Expected :com.arhohuttunen.junit5.assertions.Person@eec5a4a
Actual   :com.arhohuttunen.junit5.assertions.Person@2b2948e2
```

The Java compiler internally calls the `toString()` method to get a string representation of the object.
The problem is that the default implementation displays the hashcode value of the object.

We can make this more readable by overriding the `toString()` method in the `Person` class:

```java
@Override
public String toString() {
    return "Person{" +
            "firstName='" + firstName + '\'' +
            ", lastName='" + lastName + '\'' +
            '}';
}
```

Re-running the test we can now see the actual fields of the class:

```bash
org.opentest4j.AssertionFailedError: expected: com.arhohuttunen.junit5.assertions.Person@4b4523f8<Person{firstName='John', lastName='Doe'}> but was: com.arhohuttunen.junit5.assertions.Person@731a74c<Person{firstName='John', lastName='Doe'}>
Expected :Person{firstName='John', lastName='Doe'}
Actual   :Person{firstName='John', lastName='Doe'}
```

The expected and actual objects have the same field values, but `assertEquals()` still fails.
What is going on?

The reason is that Java object equality uses the `equals()` method for comparison.
The default implementation of the `equals()` method checks if two object references refer to the same object.
The assertion fails because of this.

{{% callout note %}}
**Additional reading:**

:bookmark: [Java hashCode() and equals()](https://howtodoinjava.com/java/basics/java-hashcode-equals-methods/)
{{% /callout %}}

To solve this problem, we need to add our implementation of `equals()` that compares the class fields.
If we override the `equals()` method, we also have to override `hashCode()`:

```java
public class Person {

    // ...

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return firstName.equals(person.firstName) &&
                lastName.equals(person.lastName);
    }

    @Override
    public int hashCode() {
        return Objects.hash(firstName, lastName);
    }
}
```

Re-running the test, we can see that it now passes.
Overriding the `equals()` method now compares objects by their fields for equality.

### Objects by Reference

Sometimes we need to make sure if two objects are referring to the same instance or not.
For example, we might need to make sure a method does not return some object but a copy of it.
JUnit 5 has the methods `assertSame()` and `assertNotSame()` for this:

```java
@Test
void personsAreNotSameInstance() {
    Person john = new Person("John", "Doe");
    Person doe = new Person("John", "Doe");

    assertNotSame(john, doe);
}
```

The example will pass as we would expect even though the two objects have the same values because they are two separate instances.
If the assertion were to fail we would get an error message:

```bash
org.opentest4j.AssertionFailedError: expected: not same but was: <Person{firstName='John', lastName='Doe'}>
Expected :not same
Actual   :<Person{firstName='John', lastName='Doe'}>
```

### Exceptions

To ensure our error handling works correctly, we can verify that a piece of code throws a specific exception under certain conditions.
This can be done with the `assertThrows()` method in JUnit 5:

```java
@Test
void divideByZeroThrowsIllegalArgumentException() {
    Calculator calculator = new Calculator();
    assertThrows(IllegalArgumentException.class, () -> calculator.divide(1, 0));
}
```

In this example, the implementation will throw `IllegalArgumentException` if we try to divide by zero.

If it doesn't throw an exception, the test will fail with an error message:

```bash
org.opentest4j.AssertionFailedError: Expected java.lang.IllegalArgumentException to be thrown, but nothing was thrown.
```

Also, if it throws an unexpected exception, the test will fail with a different error message:

```bash
org.opentest4j.AssertionFailedError: Unexpected exception type thrown ==>
Expected :<java.lang.IllegalArgumentException>
Actual   :<java.lang.ArithmeticException>
```

In some cases, we want to verify information about the exception, such as the error message or the cause.
In such cases we can capture the thrown exception:

```java
@Test
void divideByZeroThrowsIllegalArgumentException() {
    Calculator calculator = new Calculator();
    Throwable thrown = assertThrows(IllegalArgumentException.class, () -> calculator.divide(1, 0));
    assertEquals("Cannot divide by zero", thrown.getMessage());
}
```

### Timeouts

Sometimes we might want to make sure the execution time does not exceed a limit.
We can use either the `assertTimeout()` or the `assertTimeoutPreemptively()` method to do this.

The difference between these two methods is that `assertTimeout()` runs in the same thread as the code that calls it, and it won't abort if it exceeds the timeout.
On the other hand, the `assertTimeoutPreemptively()` method executes in a different thread and aborts if it exceeds the timeout.

The previous means that the first test will keep executing as long as it takes, while the second one will stop if it exceeds the timeout value. 

Let's take a look at an example:

```java
@Test
void returnValueBeforeTimeoutExceeded() {
    final String message = assertTimeout(Duration.ofMillis(50), () -> {
        Thread.sleep(100);
        return "a message";
    });
    assertEquals("a message", message);
}
```

Since the execution time will exceed the timeout, we will see an error message:

```bash
org.opentest4j.AssertionFailedError: execution exceeded timeout of 100 ms by 50 ms
```

If we want to abort the execution, we need to call the `assertTimeoutPreemptively()` method instead:

```java
@Test
void abortWhenTimeoutExceeded() {
    final String message = assertTimeoutPreemptively(Duration.ofMillis(50), () -> {
        Thread.sleep(100);
        return "another message";
    });
    assertEquals("another message", message);
}
```

If the execution time were to exceed the timeout, we would see a slightly different error message:

```bash
org.opentest4j.AssertionFailedError: execution timed out after 50 ms
```

The difference here is that the execution stopped at the timeout value.

## Custom Error Messages

Providing custom error messages for JUnit 5 assertions is easy.
All the assertions have an optional error message as the last parameter:

```java
@Test
void addNumbers() {
    Calculator calculator = new Calculator();
    assertEquals(3, calculator.add(1, 2), "1 + 2 should equal 3");
}
```

The custom error message does not replace the default error message.
Instead, assertion failure prepends the custom message to the error message:

```bash
org.opentest4j.AssertionFailedError: 1 + 2 should equal 3 ==>
Expected :3
Actual   :-1
```

In some cases, we might need to construct a little more complex error message.
In such case we can pass the error message as the last parameter in a lambda expression:

```java
@Test
void addingEmployeesToPersonnel() {
    Person employee = new Person("John", "Doe");

    Set<Person> personnel = new HashSet<>();
    personnel.add(employee);

    assertTrue(personnel.contains(employee),
            () -> String.format("Personnel file for %s was not found", employee));
}
```

The error message in the example is not that complex.
However, using this approach JUnit 5 will only construct the error message when the assertion fails.
This way, we only pay the cost if failure happens.

## Grouped Assertions

When running tests, test execution will stop at the first assertion failure.
Using JUnit 5 grouped assertions, we can run all the assertions before reporting a failure.
We can do this by using the `assertAll()` method and providing the different assertions as parameters to the method.

Let's say we want to verify that a person has a correct name.
This means that we need to assert that both the first and last name are correct:

```java
@Test
void firstAndLastNameMatches() {
    Person person = new Person("John", "Doe");

    assertAll("person"
            () -> assertEquals("John", person.getFirstName()),
            () -> assertEquals("Doe", person.getLastName())
    );
}
```

The first parameter of the `assertAll()` method is an optional title message that identifies the asserted state.

If this example were to fail, both assertions be execute before failing the test and reporting the failures together:

```bash
org.opentest4j.MultipleFailuresError: person (2 failures)
	expected: <John> but was: <Jane>
	expected: <Doe> but was: <Woodlawn>
```

As we can see, it's reporting both the failures, making it easier to fix the error.

{{% callout note %}}
A test should have only one reason to fail.
We should not try to reduce the number of tests by verifying several conditions in a single test.
However, in some cases, we might want to have more than one assertion in a test when the assertions are closely and semantically related.
{{% /callout %}}

## Advanced Matching

While the JUnit 5 assertions are sufficient for many testing scenarios, sometimes we need more powerful options.
For example, maybe we want to make sure a list has a certain size.
Or maybe we need to know if the list contains an item with a specific field value.
Or maybe we would like to verify that a list is sorted and has exact items in it
We could write some logic ourselves, but it would be better if the assertion library would do this for us. 

Here JUnit 5 assertions fall short.
Therefore, the JUnit 5 documentation recommends using third-party assertion libraries in such cases.
The most popular ones are Hamcrest, AssertJ, and Truth.

We are not going to all details about these libraries in this tutorial.
However, let's take a quick look at how some assertions might look like with each of them.

### Hamcrest

Hamcrest is the oldest one of the bunch. 
In the following example, we want to make sure a list has just one item.
We could write this with JUnit 5 assertions:

```java
    @Test
    void listHasOneItem() {
        List<String> list = new ArrayList();
        list.add("Hello");
        assertEquals(list.size(), 1);
    }
```

It's not that bad.
Let's look at the Hamcrest alternative.
We can write assertions by passing the assertion method a matcher method as an argument:

```java
    @Test
    void listHasOneItem() {
        List<String> list = new ArrayList();
        list.add("Hello");
        assertThat(list, hasSize(1));
    }
```

Reading out the assertion, this is more fluent and closer to natural language.
However, we could argue that the first example is readable enough, so maybe we are not convinced yet.

### AssertJ

Next, let's take a quick peek at AssertJ.
The main difference between Hamcrest and AssertJ is that Hacmrest relies on matcher methods, while in AssertJ we can chain the method calls.

What if we want to know if a list contains an item with a specific field value? Let's write a test using JUnit 5 only:

```java
    @Test
    void listHasPerson() {
        List<Person> people = new ArrayList<>();
        people.add(new Person("John", "Doe"));
        people.add(new Person("Jane", "Doe"));
        assertTrue(people.stream().anyMatch(p -> p.getFirstName().equals("John")));
    }
```

Ugh, that does not look pretty.
Also, what if we made an error in our logic?

Let's see how this would look like with AssertJ assertions:

```java
    @Test
    void listHasPerson() {
        List<Person> people = new ArrayList<>();
        people.add(new Person("John", "Doe"));
        people.add(new Person("Jane", "Doe"));
        assertThat(people).extracting("firstName").contains("John");
    }
```

It's quite easy to see that this is much more readable now.
We also removed logic from the test code, which can be prone to errors.

### Truth

Finally, let's check out Truth.
Truth is very similar to AssertJ.
The most significant difference is that Truth tries to provide a more straightforward API, while AssertJ has a more comprehensive set of assertions. 

Let's take a look at our third example.
We want to verify that a list is sorted and has exact items in it
This is how it would look like using Truth:

```java
    @Test
    void listHasItemsInOrder() {
        List<String> fruits = new ArrayList<>();
        fruits.add("Citron");
        fruits.add("Orange");
        fruits.add("Grapefruit");
        assertThat(fruits).containsExactly("Citron", "Grapefruit", "Orange").inOrder();
    }
``` 

Once again, this is very concise and readable.

## Summary

JUnit 5 assertions make it easier to verify that the expected test results match the actual results.

- The JUnit 5 assertions are static methods in the `org.junit.jupiter.api.Assertions` class.
- Failing assertions display the expected and actual values in their error messages.
- To provide more information about a failure, we can pass a custom error message to each assertion method.
- Using the `assertAll()` method groups all assertions, executes them, and reports failures together.
- For more complex assertions, the JUnit 5 documentation recommends using third-party assertion libraries, such as Hamcrest, AssertJ or Truth.

The example code for this guide can be found in [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-assertions).
