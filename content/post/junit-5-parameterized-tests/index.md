---
title: "A More Practical Guide to JUnit 5 Parameterized Tests"
date: 2019-05-25
author: Arho Huttunen
summary: This tutorial teaches you how to write JUnit 5 parameterized tests. It is structured so that it also answers most asked questions about parameterized tests.
categories:
  - Testing
tags:
  - JUnit 5
featured: true
image:
  focal_point: center
  preview_only: true
---

{{< youtube 0xSCbTYAiF0 >}}
<br/>

This tutorial teaches you how to write JUnit 5 parameterized tests. It is structured so that it also answers most asked questions about parameterized tests.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

{{% toc %}}

## Overview

Parameterized tests make it possible to run the same test multiple times with different arguments.
This way, you can quickly verify various conditions without writing a test for each case.

We can write JUnit 5 parameterized tests just like regular JUnit 5 tests but have to use the `@ParameterizedTest` annotation instead.
We will also have to declare an argument source for the test.
We declare these argument sources with different argument source annotations.

## Do you only need one argument?

The simplest argument source is the `@ValueSource` argument source.
It lets you specify an array of literals of primitive types (either `short`, `byte`, `int`, `long`, `float`, `double`, `char`, `boolean`, `String`, or `Class`).

Here is an example of using different strings as the test argument.

```java
@ParameterizedTest
@ValueSource(strings = { "level", "madam", "saippuakivikauppias" })
void palindromeReadsSameBackward(String string) {
    assertTrue(StringUtils.isPalindrome(string));
}
```

By the way, *saippuakivikauppias* means *soapstone vendor* in Finnish.

When we run the test, we can see from the output that the test method executed three times with different values of the string.

```sh
palindromeReadsSameBackward(String)
├─ [1] level
├─ [2] madam
└─ [3] saippuakivikauppias
```

Here is another example, where we provide `int`s to our parameterized test.

```java
@ParameterizedTest
@ValueSource(ints = { 3, 6, 15})
void divisibleByThree(int number) {
    assertEquals(0, number % 3);
}
```

Another source of a single argument is the `@EnumSource` annotation.
The annotation takes an enum type as an argument and provides the test with the enum constants.

For example:

```java
enum Protocol {
    HTTP_1_0, HTTP_1_1, HTTP_2
}
```

```java
@ParameterizedTest
@EnumSource(Protocol.class)
void postRequestWithDifferentProtocols(Protocol protocol) {
    webServer.postRequest(protocol);
}
```

If we run the test, we can see that the test has been executed once per every value of the `Protocol` enumeration.

## What about null values?

The `@ValueSource` annotation doesn't accept null values.

There is one special annotation called `@NullSource` that will provide a null argument for the test.
Another special annotation is `@EmptySource`, which provides an empty value for either a `String`, `List`, `Set`, `Map`, or an array.

In this example, we are passing the values of null, an empty string, and a blank string to the test method.

```java
@ParameterizedTest
@NullSource
@EmptySource
@ValueSource(strings = { " " })
void nullEmptyAndBlankStrings(String text) {
    assertTrue(text == null || text.trim().isEmpty());
}
```

It is also possible to use `@NullAndEmptySource` which combines the two.

## Do you need multiple arguments?

The `@ValueSource` and `@EnumSource` annotations only work when our test method takes one argument.
However, we often need more than that.

`@MethodSource` allows you to refer to a factory method that returns the arguments.
Such methods must return a `Stream`, `Iterable`, `Iterator`, or an array of arguments.

Let's assume that we have a `DateUtils` class that gets the name of the month for a number.
We need to pass multiple parameters in our parameterized test, so we can use a `Stream` of `Arguments` in our factory method.

```java
@ParameterizedTest
@MethodSource("numberToMonth")
void monthNames(int month, String name) {
    assertEquals(name, DateUtils.getMonthName(month));
}

private static Stream<Arguments> numberToMonth() {
    return Stream.of(
            arguments(1, "January"),
            arguments(2, "February"),
            arguments(12, "December")
    );
}
```

When we refer the factory method in `@MethodSource`, it will then provide the test with different values of `month` and `name` parameters.

If we don't provide a method name to the `@MethodSource` annotation, JUnit 5 will try to find a method with the same name instead.

```java
@ParameterizedTest
@MethodSource
void monthNames(int month, String name) {
    assertEquals(name, DateUtils.getMonthName(month));
}

private static Stream<Arguments> monthNames() {
    return Stream.of(
            arguments(1, "January"),
            arguments(2, "February"),
            arguments(12, "December")
    );
}
```

## Do you want to share your argument providers between tests?

It is possible to refer to a method in another class in `@MethodSource`.
We have to use the fully qualified name of the method name to do so.

```java
package com.arhohuttunen;

import java.util.stream.Stream;

public class StringsProvider {
    private static Stream<String> palindromes() {
        return Stream.of("level", "madam", "saippuakivikauppias");
    }
}
```

The fully qualified name is a combination of the package name, class name, and the method name.

```java
@ParameterizedTest
@MethodSource("com.arhohuttunen.StringsProvider#palindromes")
void externalPalindromeMethodSource(String string) {
    assertTrue(StringUtils.isPalindrome(string));
}
```

Another option for sharing is to write a custom class that implements the `ArgumentsProvider` interface.

```java
public class PalindromesProvider implements ArgumentsProvider {
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
        return Stream.of("level", "madam", "saippuakivikauppias").map(Arguments::of);
    }
}
```

This can be then specified in the test with the `@ArgumentsSource` annotation.

```java
@ParameterizedTest
@ArgumentsSource(PalindromesProvider.class)
void externalPalindromeMethodSource(String string) {
    assertTrue(StringUtils.isPalindrome(string));
}
```

## Do you have a lot of data?

The `@CsvSource` annotation allows you to use a list of comma-separated string values.
Using the annotation makes it possible to provide multiple parameters to the test method in quite a compact way.

```java
@CsvSource({
        "Write a blog post, IN_PROGRESS, 2020-12-20",
        "Wash the car, OPENED, 2020-12-15"
})
void readTasks(String title, Status status, LocalDate date) {
    System.out.printf("%s, %s, %s", title, status, date);
}
```

If we write a lot of test data in the test code, the test quickly becomes unreadable.
One solution is to provide the data in an external CSV file using the `@CsvFileSource` annotation.

Using the previous example, we start by creating a comma-separated list of parameters in `tasks.csv` file that we will put in `src/test/resources`.
Each line from the file works as a list of parameters.

```
Write a blog post, IN_PROGRESS, 2020-12-20
Wash the car, OPENED, 2020-12-15
```

Next, we use the `@CsvFileSource` annotation to provide the test method with the data.

```java
@ParameterizedTest
@CsvFileSource(resources = "/tasks.csv")
void readTasks(String title, Status status, LocalDate date) {
    System.out.printf("%s, %s, %s", title, status, date);
}
```

### How to provide empty CSV arguments?

If `@CsvSource` has an empty value, JUnit 5 will always treat it as `null`.

```java
@ParameterizedTest
@CsvSource(", IN_PROGRESS, 2020-12-20")
void nullArgument(String title, Status status, LocalDate date) {
    assertNull(title);
}
```

The string needs to be quoted with single quotes, so that it becomes an empty string.

```java
@ParameterizedTest
@CsvSource("'', IN_PROGRESS, 2020-12-20")
void emptyArgument(String title, Status status, LocalDate date) {
    assertTrue(title.isEmpty());
}
```

If we would like to replace some specific string with null values, we can use the `nullValues` argument of `@CsvSource`.

```java
@ParameterizedTest
@CsvSource(value = "NULL, IN_PROGRESS, 2020-12-20", nullValues = "NULL")
void customNullArgument(String title, Status status, LocalDate date) {
    assertNull(title);
}
```

### How to convert strings into different types?

To better support use cases like `@CsvSource`, JUnit 5 does automatic argument conversion for primitive types, enums, and the date and time types from the `java.time` package.
The conversion depends on the type of each method parameter.

This means, for example, that it automatically converts the following date strings to `LocalDate` instances.

```java
@ParameterizedTest
@ValueSource(strings = { "2018-01-01", "2018-01-31" })
void convertStringToLocalDate(LocalDate localDate) {
    assertEquals(Month.JANUARY, localDate.getMonth());
}
```

JUnit 5 parameterized tests support many more types by default.
Instead of going through all of them here, we can check the [JUnit 5 implicit conversion](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests-argument-conversion-implicit) documentation for a list of target types.

If JUnit 5 is not able to convert the argument, it will try to call either of the two in the target type:

1. A constructor with a single `String` argument
2. A `static` method accepting a single `String` argument, which returns an instance of the target type 

In the following example, JUnit 5 will call the constructor of `Person` to do the type conversion from `String`.

```java
public class Person {
    private String name;

    public Person(String name) {
        this.name = name;
    }
}
```

```java
@ParameterizedTest
@CsvSource("John Doe")
void fallbackStringConversion(Person person) {
    assertEquals("John Doe", person.getName());
}
```

Consequently, the implementation of that `Person` class in the next example would work as well.

```java
public class Person {
    private final String name;

    public static fromName(String name) {
        return new Person(name);
    }
}
```

### How to convert your custom types?

If we need to write a custom argument converter, we need to implement the `ArgumentConverter` interface.
We can then annotate any parameters needing custom conversion with the `@ConvertWith` annotation.

For example's sake, let's write an argument converter that converts hex values into decimal values.
Instead of implementing `ArgumentConverter`, we can also extend `TypedArgumentConverter` if the converter only handles one type.

```java
class HexConverter extends TypedArgumentConverter<String, Integer> {
    protected HexConverter() {
        super(String.class, Integer.class);
    }

    @Override
    public Integer convert(String source) throws ArgumentConversionException {
        try {
            return Integer.parseInt(source, 16);
        } catch (NumberFormatException e) {
            throw new ArgumentConversionException("Cannot convert hex value", e);
        }
    }
}
```

Next, in our test, we need to annotate the parameter that needs custom conversion with `@ConvertWith`.

```java
@ParameterizedTest
@CsvSource({
        "15, F",
        "16, 10",
        "233, E9"
})
void convertWithCustomHexConverter(int decimal, @ConvertWith(HexConverter.class) int hex) {
    assertEquals(decimal, hex);
}
```

To make the test itself a little less technical and more readable, we can further create a meta-annotation that wraps the conversion.

```java
@Target({ ElementType.ANNOTATION_TYPE, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@ConvertWith(HexConverter.class)
public @interface HexValue {
}
```

Now we can use our new composed annotation to make the test more readable.

```java
@ParameterizedTest
@CsvSource({
        "15, F",
        "16, 10",
        "233, E9"
})
void convertWithCustomHexConverter(int decimal, @HexValue int hex) {
    assertEquals(decimal, hex);
}
```

### How to convert multiple arguments into a single object?

By default, arguments provided to a parameterized test correspond to a single method parameter.
It is possible to aggregate these arguments into a single test method argument using a `ArgumentsAccessor`.

To create a more readable and reusable arguments aggregator, we can write our own.

```java
public class TaskAggregator implements ArgumentsAggregator {
    @Override
    public Object aggregateArguments(
            ArgumentsAccessor accessor,
            ParameterContext context
    ) throws ArgumentsAggregationException {

        return new Task(
                accessor.getString(0),
                accessor.get(1, Status.class),
                accessor.get(2, LocalDate.class)
        );
    }
}
```

```java
@ParameterizedTest
@CsvSource({
        "Write a blog post, IN_PROGRESS, 2020-12-20",
        "Wash the car, OPENED, 2020-12-15"
})
void aggregateArgumentsWithAggregator(@AggregateWith(TaskAggregator.class) Task task) {
    System.out.println(task);
}
```

Just like with the custom argument converter, we can also create a shorthand annotation for the aggregator.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@AggregateWith(TaskAggregator.class)
public @interface CsvToTask {
}
```

```java
@ParameterizedTest
@CsvSource({
        "Write a blog post, IN_PROGRESS, 2020-12-20",
        "Wash the car, OPENED, 2020-12-15"
})
void aggregateArgumentsWithAnnotation(@CsvToTask Task task) {
    System.out.println(task);
}
```

Now we can use the aggregator annotation anywhere we want to.

## How to customize display names?

By default, JUnit 5 parameterized tests' display names include the invocation index and String representation of all the parameters.
However, we can customize the display name via the name attribute of the `@ParameterizedTest` annotation.

Let's take a look at the previous month names example again.

```java
@ParameterizedTest(name = "{index} => number={0}, month={1}")
@MethodSource
void monthNames(int month, String name) {
    assertEquals(name, DateUtils.getMonthName(month));
}

private static Stream<Arguments> monthNames() {
    return Stream.of(
            arguments(1, "January"),
            arguments(2, "February"),
            arguments(12, "December")
    );
}
```

The name attribute holds placeholders for `{index}` that is the index of the current test invocation and `{0}`, `{1}`, … that is the actual parameter value.

Now when we run the test we get output similar to this:

```
 monthNames(int, String)
├─ 1 => number=1, month=January
├─ 2 => number=2, month=February
└─ 3 => number=12, month=December
```

## Summary

JUnit 5 parameterized tests allow you to remove duplication from test code.
They make it possible to execute the same test several times using different inputs.

- Using `@ValueSource` is enough in most cases when you have just one argument, but you can also use `@EnumSource`, `@NullSource` and `@EmptySource`
- If there are multiple arguments `@MethodSource` is the right choice in most cases, and you can reuse providers with `@ArgumentsSource` 
- For data-driven tests you can use `@CsvFileSource`
- You can write your custom argument conversions with `ArgumentConverter`
- To customize arguments aggregation you can use `ArgumentsAggregator`
  
The example code for this guide can be found in [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-parameterized-tests).
