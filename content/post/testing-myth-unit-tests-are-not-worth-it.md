---
title: "Testing Myth #2: Unit Tests Are Not Worth It"
date: 2016-02-01
authors:
  - arhohuttunen
categories:
  - Software Craft
tags:
  - testing
image:
  filename: stock/home-on-building-blocks-surrounded-by-ruins-of-house-that-collapsed.jpg
---

There are a lot of developers and managers who think that writing unit tests is just extra work. Suggesting that we should write more unit tests seems to receive ill responses. I think there are many people out there who still don't understand the purpose of unit testing.

This kind of thinking is probably the result of following kind of experiences:

- Writing unit tests is really hard and time consuming.
- Even small changes in requirements keep breaking the unit tests.
- Unit tests are not finding any real bugs.

It is not that writing unit tests is somehow fundamentally laborious. These kind of experiences are symptoms of something else.

## Serving a purpose

Unit testing is not really about testing or finding bugs. <strong>Unit testing is a design activity and not a testing activity</strong>, especially when practicing test-driven development.

The two most important benefits from unit testing are:

1. Getting a loosely [coupled](https://en.wikipedia.org/wiki/Coupling_(computer_programming)) design with high [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science)). This makes it easier to make changes to existing code.
2. Providing a fast automated regression test suite. This facilitates change by reducing the fear to refactor.

Unit testing makes you use the class you are creating. If your class gets too big or you add too many dependencies you get punished by extra work writing the test. This in turn **drives your design to be more independent and loosely coupled**.

When classes become more independent they also become more resilient to changes. This way the improved design also makes the tests less brittle.

When you finish your implementation you end up with a nice regression test suite. These tests do not verify that the system works correctly but **make sure that the functionality does not change**.

**Most value from unit tests is gained in the act of writing tests while writing the code**. This is why adding tests afterwards may feel like not getting enough results for the effort.

## Possible missteps

Having bad experiences from unit testing can be caused by misunderstanding. Following things may attribute to thinking that unit tests are not worth writing:

- Writing integration tests and calling them unit tests.
- Writing tests after the code is written and not during or before.
- Writing tests afterwards without refactoring the code.
- Writing tests for someone else's code.

Sometimes people say that they are writing unit tests when they are really writing integration tests. If in your test setup you are creating objects that have any kind of external dependencies chances are you are really doing integration testing.

The difference is that unit testing drives the design while integration testing does not. Both are needed but it is important to understand the purpose at the given situtation.

Unit tests can be a lot of work to write afterwards. If you want to test an entire system with a decent code coverage it is going to take a lot of time.

The primary value of unit testing is improving your design. If you are going to add unit tests to existing code you will need to refactor the code to get this benefit. Otherwise you are just wasting a lot of time and money.
