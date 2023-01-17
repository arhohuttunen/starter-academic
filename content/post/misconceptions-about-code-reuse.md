---
title: Misconceptions About Code Reuse
date: 2015-06-29
authors:
  - arhohuttunen
categories:
  - Software Craft
image:
  filename: stock/mature-caucasian-man-scratching-head.jpg
---

We all have been taught that reusable code is good. We all know why we should do it but there are some misconceptions about how to do it.

There are two common misconceptions about reusable code:

- Using inheritance in order to achieve code reuse.
- Creating a static utility or helper class in order to reuse methods.

Mostly these misconceptions are the result of procedural thinking. People usually understand how certain object-oriented principles work. However, it is seldom taught what is the purpose of such principles.

## Inheritance is not the way to achieve code reuse

While writing code you might find potential methods to extract for code reuse. You might decide to pull the method up in the class hierarchy in order to make it available in all derived classes.

The other case is that you identify shared code between two or more classes and decide to create a super class in order to reuse code between the classes.

There are some problems with using inheritance for code reuse:

- The interface of the classes become easily bloated with unneeded methods.
- The methods can only be reused inside the class hierarchy.
- Methods are not easily extendable and added genericity increases complexity.
- Inheritance reduces options on sub classing as you cannot sub class something else.
- Inheritance ties you to the implementation of the super class.

A better way to achieve code reuse is to favor [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance). This means that instead of just extracting methods you should be identifying new classes. Composition keeps your options open, you can reuse different implementations and you can change your mind easily.

## What inheritance is really meant for

Inheritance is used to categorize things and allow us to use polymorphism. You can build hierarchies of concepts at different levels of abstraction. Polymorphism allows same piece of code to manage all objects in a category independent of their implementation.

The main reason to use inheritance is software design and not code reuse. Derived classes can incidentally reuse code from the super class but it is not a driving force to use inheritance.

## Utility classes are procedural programming

Utility classes are the result of trying to follow the [don't repeat yourself](https://en.wikipedia.org/wiki/Don't_repeat_yourself) practice. Utility classes are an indicator of laziness and lack of domain knowledge because it is easier to put common code into utility classes rather than trying to identify a proper place for it.

Writing utility classes is better than copy pasting code but you can do better. Let's see what is wrong with static utility classes:

- Utility classes are not object-oriented but procedural which violates [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)).
- Dumping methods that have no better place into utility classes mixes responsibilities breaking the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).
- You cannot extend static methods or implement interfaces.
- Static methods can make testing hard or using test doubles very complicated.

Instead of just dumping code as static methods in some placeholder class you should identify responsibilities. Create proper classes out of these responsibilities and call the classes by that name.

This not only puts behavior into a single place but also makes it easier to understand what the class does. It will probably lead into several smaller classes having well defined responsibilities instead of one bloated utility class.

For example, `MessageParser` and `MessageHandler` spell out clearly what they are supposed to do while `MessageUtil` does not.

Utility classes often also exist for common reasons. Chances are that you are not the only one trying to write them. It is very likely that you can use some open source libraries instead of reinventing the wheel.
