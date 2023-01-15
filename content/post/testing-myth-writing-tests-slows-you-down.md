---
title: "Testing Myth #1: Writing Tests Slows You Down"
date: 2016-01-18
authors:
  - arhohuttunen
categories:
  - Testing
tags:
  - Clean code
image:
  filename: stock/snail-race.jpg
---

Let's assume we have decided to increase the stability of our software. So we decide to write tests for our code. The problem is that the customer is requesting new features and deadlines are approaching. Testing will slow us down. We can skip them now and write them later, right?

Wrong. **Testing will not slow us down**. Also if we decide to skip writing them it's almost 100% certain that they won't be written at all.

While tests _might_ slow us down at the begin of the development cycle, the truth is **the longer the software needs to be maintained tests will dramatically speed us up**.

## False assumptions

Thinking that writing tests will slow down the overall development time includes at least two false assumptions:

- The actual coding of a feature takes the same time regardless if you are writing tests or not.
- Once a feature is complete (or rather the first version of it) there are no additional costs.

The truth is, usually what happens is that:

- If you write a test first, the actual coding time of a feature can be even less than if you did not. Just thinking about what to test forces you to identify what you actually need to do.
- There will be maintenance costs, manual testing costs and bug fixing costs that are closely related to the original implementation.

## Short term gains

Only seeing the short term effort misses a lot of points that will happen over the time. Let's see what happens in long term when you do not have any tests:

- You make a modification to a feature. The only way to test that it works is to do it manually through the UI.
- Sometimes a tester or an end-user discovers a bug. You end up debugging through the UI.
- You need to add a feature but it is really hard. The code has become a tangled mess that you are afraid to change. You add some glue and know it's a horrible hack hoping it holds together.

Debugging will become more and more difficult. Adding changes will take more time. Testing thoroughly will take more time if the code has become a big pile of mud. The economics of not having tests work so that **changing anything will become more and more costly over time**.

Having tests in the first place:

- Discover errors at the earliest stage possible, saving time from manual testing.
- Make you spend less time debugging and help pinpoint errors.
- Allow you to make changes easier and without the fear of braking everything up.

The long term effects of not having tests are quickly starting to cost more than writing the tests in the first place. The longer the code needs to be supported, the harder the activities described before become.

## There are two sides to every coin

Now let's face it, writing tests does take time. Especially if you are not familiar with writing tests. However, there once was a time when you did not know how to code either. **You will get better at writing tests only by practicing**.

Also tests that are hard to maintain will introduce an extra cost. This can be kept at control by not writing tests for the implementation but for features. **The test code needs to be as decoupled from the implementation as possible**.

Something that will dramatically affect the success of adding tests is the quality. Low quality tests will most likely introduce extra maintenance costs. **Test code quality should be at the same level with the production code**.

If there is no culture of testing and you are working in a legacy project without tests you will almost certainly see a decrease in productivity at first. It can be hard to see the benefits in the beginning. This is a cost to pay if you want to spend less time in maintenance and more time being productive.
