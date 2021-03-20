---
title: "JUnit 5 Maven Example: Running Tests with Surefire"
type: book
linktitle: Running With Maven
url: /junit-5-maven-example
date: 2018-01-03
author: Arho Huttunen
summary: Learn how to get the required dependencies for writing JUnit 5 tests with Maven. Also, learn how to configure the Maven Surefire plugin to run the tests.
categories:
  - Testing
tags:
  - JUnit 5
  - Maven
weight: 20
---

In this JUnit 5 Maven example, we will learn how to get the required dependencies for writing JUnit 5 tests with Maven. Also, we will learn how to configure the Maven Surefire plugin to run the tests.

## Required Dependencies

First, to be able to write JUnit 5 tests we need the `junit-jupiter` artifact as a dependency in `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.7.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

In addition, to be able to run JUnit 5 tests we have to add the Maven Surefire plugin:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.22.2</version>
        </plugin>
    </plugins>
</build>
```

Now we have the most basic setup for writing and running JUnit 5 tests with Maven.

### Configuration for Older Versions

Beginning from JUnit Jupiter 5.4.0 there is now an aggregator artifact `junit-jupiter` that transitively pulls in dependencies on `junit-jupiter-api`, `junit-jupiter-params`, and `junit-jupiter-engine` for simplified dependency management.

To be able to write JUnit 5 tests using an older version we need the `junit-jupiter-api` artifact as a dependency:

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.3.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Beginning from Maven Surefire 2.22.0 there is now native support for JUnit Jupiter. When using Maven Surefire 2.21.0 or older, we have to use a provider for Maven Surefire plugin to run the tests:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.21.0</version>
            <dependencies>
                <dependency>
                    <groupId>org.junit.platform</groupId>
                    <artifactId>junit-platform-surefire-provider</artifactId>
                    <version>1.3.2</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

We also have to add JUnit Jupiter test engine to the runtime classpath. We add the dependency to `maven-surefire-plugin` dependencies:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>2.21.0</version>
            <dependencies>
                <dependency>
                    <groupId>org.junit.platform</groupId>
                    <artifactId>junit-platform-surefire-provider</artifactId>
                    <version>1.3.2</version>
                </dependency>
                <dependency>
                    <groupId>org.junit.jupiter</groupId>
                    <artifactId>junit-jupiter-engine</artifactId>
                    <version>5.3.2</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

Now we have the setup for writing and running JUnit 5 tests with older versions of Maven Surefire.

## Running tests

The `maven-surefire-plugin` discovers tests under `src/test/java` directory by default.

Let’s check if our configuration works by adding a very simple test that does nothing:

```java
class MavenExampleTest {

    @Test
    void shouldRun() {

    }
}
```

Now we can run our tests on the command line with:

```
$ mvn test
```

We should see output similar to this:

```
[INFO] --- maven-surefire-plugin:2.22.0:test (default-test) @ junit5-maven ---
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running MavenExampleTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 s - in MavenExampleTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

There we go! Maven Surefire is now running our tests.

## Summary

In this JUnit 5 Maven example we have learned how to add the required dependency for writing JUnit 5 tests and how to configure Maven Surefire plugin to be able to run the tests.

The example code for this guide can be found on [GitHub][2].

[1]:	/junit-5-tutorial
[2]:	https://github.com/arhohuttunen/junit5-examples/tree/main/junit5-maven
