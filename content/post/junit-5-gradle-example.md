---
title: JUnit 5 Gradle Example
subtitle: Running Tests with Gradle
date: 2018-01-01
authors:
  - arhohuttunen
summary: Learn how to get the required dependencies for writing JUnit 5 tests with Gradle. Also, learn how to configure the JUnit Gradle plugin to run the tests.
categories:
  - Testing
tags:
  - JUnit 5
  - Gradle
image:
  filename: edited/junit-5-gradle.png
---

In this JUnit 5 Gradle example, we will learn how to get the required dependencies for writing JUnit 5 tests with Gradle. Also, we will learn how to configure the JUnit Gradle plugin to run the tests.

This article is part of the [JUnit 5 Tutorial](/junit-5-tutorial).

## Required dependencies

First, to be able to write JUnit 5 tests we need the `junit-jupiter` artifact as a dependency in `build.gradle`:

```gradle
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.8.0")
}
```

Now all we need is to tell to use the JUnit platform in the tests:

```gradle
test {
    useJUnitPlatform()
}
```

Now we have the most basic setup for writing and running JUnit 5 tests with Gradle.

### Configuration for Older Versions

Beginning from JUnit Jupiter 5.4.0 there is now an aggregator artifact `junit-jupiter` that transitively pulls in dependencies on `junit-jupiter-api`, `junit-jupiter-params`, and `junit-jupiter-engine` for simplified dependency management. This means that we don't need additional dependencies to be able to write parameterized tests.

To be able to write JUnit 5 tests using an older version we need the `junit-jupiter-api` artifact as a dependency. We also need the JUnit Jupiter test engine in the test runtime classpath:

```gradle
dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter-api:5.3.2")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine:5.3.2")    
}
```

Beginning from Gradle 4.6 there is now native support for JUnit Jupiter. When using Gradle 4.5 or older, to be able to run JUnit 5 tests we have to configure the JUnit Gradle plugin:

```gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.junit.platform:junit-platform-gradle-plugin:1.3.2'
    }
}

apply plugin: 'org.junit.platform.gradle.plugin'
```

Now we have the setup for writing and running JUnit 5 tests with older versions of Gradle.

## Running tests

The JUnit Gradle plugin discovers tests under `src/test/java` directory by default.

Let’s check if our configuration works by adding a very simple test that does nothing:

```java
class GradleExampleTest {

    @Test
    void shouldRun() {

    }
}
```

Now we can run our tests on the command line with:

```
$ gradle test
```

We should see output similar to this:

```
:test
BUILD SUCCESSFUL in 2s
```

There we go! JUnit Gradle plugin is now running our tests.

## Summary

In this JUnit 5 Gradle example we have learned how to add the required dependency for writing JUnit 5 tests and how to configure JUnit Gradle plugin to be able to run the tests.

The example code for this guide can be found on [GitHub](https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-gradle).
