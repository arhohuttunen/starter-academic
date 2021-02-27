---
title: JUnit 5 with Kotlin for Java Developers
subtitle: Crash course for converting your JUnit 5 Java Tests into Kotlin
date: 2019-04-14
author: Arho Huttunen
summary: Learn the differences between writing JUnit 5 tests in Kotlin and Java. Learn to configure JUnit 5 in the build script using Gradle Kotlin DSL.
categories:
  - Testing
tags:
  - JUnit 5
  - Kotlin
featured: true
image:
  focal_point: center
  preview_only: true
---

{{< youtube 94KTm9b00VM >}}
<br/>

In this article, we will learn the differences between writing JUnit 5 tests in Kotlin and Java. We will also learn how to configure JUnit 5 in our build script using the Gradle Kotlin DSL.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

{{% toc %}}

## Configuration

We write traditional Gradle build scripts as `build.gradle` files using the Groovy DSL. Gradle's Kotlin DSL provides an alternative syntax for several improvements, such as better content assist and refactoring. The build script written in Kotlin DSL is named `build.gradle.kts`.

To write JUnit 5 tests in Kotlin, we first need the `junit-jupiter` artifact as a dependency in `build.gradle.kts` file and need to tell to use JUnit platform in the tests.

```gradle
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.7.0")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

## Basic Functionality

Most of the JUnit 5 functionality works in Kotlin just like it would in Java. Everything works out of the box.

One notable difference is how we can customize the display names of methods displayed in test reports or IDEs.

In Java, we could use the `@DisplayName` annotation to make the method name readable. However, in Kotlin, we can use backtick identifiers for variables and methods.

```kotlin
@Test
fun `1 + 2 = 3`() {
    assertEquals(3, calculator.add(1, 2))
}
```

Using backticks makes both the code and the test results more readable. Usually, we should not use such method names, but it is quite handy for this purpose.

## Lazy Evaluation

JUnit 5 added lazy evaluation of error message using lambdas. Using lambdas avoids constructing expensive error messages unnecessarily.

In Kotlin, if the last parameter of a function accepts a function, it can be rewritten as a lambda expression outside the parentheses.

```kotlin
@Test
fun `1 + 2 = 3`() {
    assertEquals(3, calculator.add(1, 2)) {
        "1 + 2 should equal 3"
    }
}
```

Compared to Java, this makes the syntax even cleaner when writing assertions in Kotlin.

## Assertions

Any JUnit 5 assertions in Java work in Kotlin as well. However, there are a couple of Kotlin specific assertion methods that are more suitable for the language. These assertion methods are top-level functions in the `org.junit.jupiter.api` package.

Here is an example of asserting that the code throws an exception. In Java, we would pass a lambda inside of the `assertThrows()` call. In Kotlin, we can again make this more readable by adding a lambda after the assertion call.

```kotlin
@Test
fun `Divide by zero should throw ArithmeticException`() {
    assertThrows<ArithmeticException> {
        calculator.divide(1, 0)
    }
}
```

The simpler lambda syntax applies to grouped assertions as well. Grouped assertions make it possible to perform multiple assertions at a time and report failures together.

Like in Java, we can write lambdas inside the `assertAll()` call in Kotlin, but the syntax is less verbose.

```kotlin
@Test
fun `Square of a number should equal the number multiplied by itself`() {
    assertAll(
            { assertEquals(1, calculator.square(1)) },
            { assertEquals(4, calculator.square(2)) },
            { assertEquals(9, calculator.square(3)) }
    )
}
```

Compared to Java this is once again a little less verbose and more readable.

## Parameterized Tests

There are several ways to write [JUnit 5 parameterized tests](/junit-5-parameterized-tests/). Most of these approaches work without any changes in Kotlin as well.

With that in mind, there is a difference when using the `@MethodSource` annotation. The annotation expects a static method inside the class as the source of the parameters.

To achieve the same in Kotlin, we have to create a companion object and annotate the method with `@JvmStatic`. The annotation will make the method exist as a Java static method.

```kotlin
companion object {
    @JvmStatic
    fun squares() = listOf(
            Arguments.of(1, 1),
            Arguments.of(2, 4),
            Arguments.of(3, 9)
    )
}

@ParameterizedTest(name = "Square of {0} should equal {1}")
@MethodSource("squares")
fun `Square of a number`(input: Int, expected: Int) {
    assertEquals(expected, calculator.square(input))
}
```

If we didn't annotate the method with `@JvmStatic`, we would get the following error.

```bash
org.junit.platform.commons.JUnitException: Could not find method [squares] in class [com.arhohuttunen.junit5.kotlin.CalculatorParameterizedTest]
```

Using parameterized tests like this is ok but not as convenient as in Java. Another thing to note is that there can be only one companion object per class, so all parameter-providing methods need to be together.

## Dynamic Tests

JUnit 5 introduces a new programming model that allows us to generate _dynamic tests_ at runtime by a factory method annotated by the `@TestFactory` annotation.

Typically we would provide a list of `DynamicTest` instances. Let's use the calculator from previous examples.

```kotlin
@TestFactory
fun `Square of a number`() = listOf(
        DynamicTest.dynamicTest("Square of 1 should equal 1") {
            assertEquals(1, calculator.square(1))
        },
        DynamicTest.dynamicTest("Square of 2 should equal 4") {
            assertEquals(4, calculator.square(2))
        },
        DynamicTest.dynamicTest("Square of 3 should equal 9") {
            assertEquals(9, calculator.square(3))
        }
)
```

Each of the dynamic tests would appear as their own tests. However, this is not very clean and has some duplication.

Yet again, there is a way to make this more readable. We can use some functional mapping to remove the duplication.

```kotlin
@TestFactory
fun `Square of a number`() = listOf(
        1 to 1,
        2 to 4,
        3 to 9
).map { (input, expected) ->
    DynamicTest.dynamicTest("Square of $input should equal $expected") {
        assertEquals(expected, calculator.square(input))
    }
}
```

The functional approach is pretty close to what we did with parameterized tests but with a slightly different syntax.

## Nested tests

Nested tests in JUnit 5 allow us to define a hierarchical structure for the tests. Coming from Java, we might expect the following example to work.

```kotlin
class NestedTest {
    @Nested
    class GetRequest {
        @Test
        fun `return existing entity`() {}
    }

    @Nested
    class PostRequest {
        @Test
        fun `create new entity`() {}
    }
}
```

Such an example will give us a warning _Only non-static nested classes can serve as @Nested test classes_. JUnit 5 will not discover any tests for execution.

By default, a nested class in Kotlin is similar to a static class in Java. Only non-static nested classes (i.e. inner classes) can serve as `@Nested` test classes.

The solution is to mark the class as an `inner class` in Kotlin.

```kotlin
class NestedTest {
    @Nested
    inner class GetRequest {
        @Test
        fun `return existing entity`() {}
    }

    @Nested
    inner class PostRequest {
        @Test
        fun `create new entity`() {}
    }
}
```

Now JUnit 5 will discover the inner nested test classes and their tests.

## Static methods and fields

We already briefly touched on static methods and Kotlin. To make a Kotlin method visible as a Java static method, we have to create a companion object and annotate the method with `@JvmStatic`.

```kotlin
companion object {
    @JvmStatic
    fun squares() = listOf(
            Arguments.of(1, 1),
            Arguments.of(2, 4),
            Arguments.of(3, 9)
    )
}
```

Another possible pitfall is when we have to use static fields. In Java, you make the field `static`, so if you are new to Kotlin, you would expect something like this to work.

```kotlin
class RegisterStaticExtensionTest {
    companion object {
        @RegisterExtension
        val jettyExtension: JettyExtension = JettyExtension()
    }
}
``` 

However, if we write code like this, we will see an error about the field being private:

```bash
org.junit.platform.commons.PreconditionViolationException: Failed to register extension via @RegisterExtension field [private static final com.arhohuttunen.junit5.kotlin.JettyExtension com.arhohuttunen.junit5.kotlin.RegisterStaticExtensionTest.jettyExtension]: field must not be private.
```

The solution is to annotate the field with `@JvmField`.

```kotlin
class RegisterStaticExtensionTest {
    companion object {
        @JvmField
        @RegisterExtension
        val jettyExtension: JettyExtension = JettyExtension()
    }
```

Adding the annotation will expose the Kotlin property as a Java field, and JUnit 5 will now see the field.

## Lifecycle Methods

The [JUnit 5 lifecycle methods](/junit-5-test-lifecycle/) all work in Kotlin as well.

However, the methods annotated with `@BeforeAll` and `@AfterAll` need to be static by default. The reason is that JUnit 5 creates a new test instance per test method, and there is no other way to share state between all tests.

Luckily, it is possible to create a test instance per class instead in JUnit 5 by annotating the test class with `@TestInstance(Lifecycle.PER_CLASS)`. The lifecycle change removes the requirement for static methods.

```kotlin
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class LifecycleTest {
    @BeforeAll
    fun beforeAll() {
        println("Before all")
    }

    @AfterAll
    fun afterAll() {
        println("After all")
    }

    @Test
    fun firstTest() {
        println("First test")
    }

    @Test
    fun secondTest() {
        println("Second test")
    }
}
``` 

Since this now shares instance state between tests, you might need to reset state in `@BeforeEach` or `@AfterEach`, if your test methods rely on state stored in instance variables. In general, try to avoid writing tests that depend on such state. 

## Repeatable Annotations

Currently, Kotlin does not support repeatable annotations. Consequently, using multiple extensions or tags for a test is slightly more complicated than in Java.

For example, in Java we can repeat the `@Tag` annotation to give multiple tags to a test.

```java
@Tag("first")
@Tag("second")
class RepeatableAnnotationTest {}
```

In Kotlin, however, we cannot have multiple `@Tag` annotations. Instead, we have to use a `@Tags` annotation to wrap the repeated tags.

```kotlin
@Tags(
        Tag("first"),
        Tag("second")
)
class RepeatableAnnotationTest
```

The same happens with multiple extensions, so we cannot use multiple `@ExtendWith` annotations directly. Instead, we need to wrap the extensions with an `@Extensions` annotation.

```kotlin
@Extensions(
        ExtendWith(CoolestEverExtension::class),
        ExtendWith(SecondBestExtension::class)
)
class RepeatableAnnotationTest
```

## Summary

Most of the JUnit 5 features work perfectly well in Kotlin, although there are some cases where the syntax is somewhat different than in Java. However, due to the way Kotlin language works, we can often make the code more readable.

The example code for this guide can be found on [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-gradle-kotlin).
