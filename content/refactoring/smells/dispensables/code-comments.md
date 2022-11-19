---
title: Code comments
url: /code-comments/
type: book
date: 2015-05-11
author: Arho Huttunen
weight: 60
sections_weight: 60
---

Code comments are a code smell. This statement might feel contrary to reason when you have been taught to always comment your code. Having comments might however tell you that the implementation is too complex.

Don't get me wrong, I am not saying that you should not write any comments. You should however pay attention to when and why you are writing them.

## Symptoms

You should notice this smell when a method is filled with explanatory comments. Another type of this smell is redundant comments, e.g. setter or getter comments telling that the method sets or gets a value.

## Causes

Usually comments have been written with good intentions. The author noticed that the code is not obvious and added comments. If you feel the urge to add a comment, you should first try to change to code so that comments are not necessary.

## Solutions

It is pretty easy to get rid of most comments.

- If there is a complex expression needing explanation, you should split it. Use **extract variable** to create subexpressions and name them to reveal their intention.
- If there is a block of code needing explanation, use **extract method** to make it a separate method. Name the method so that it explains the same thing as the comment.
- If a method has already been extracted but comments are still needed to explain what it does, use **rename method**. Give the method a self-explanatory name.
- If you need to state some rules about the required state of the system, use **introduce assertion**.

Don't fall into the trap of not stating the obvious though. Making up good names that everyone understands can be difficult.

### Extract variable

There is an expression that is hard to understand.

```java
double price() {
  // price is base price - quantity discount + shipping
  return quantity * itemPrice -
    Math.max(0, quantity - 500) * itemPrice * 0.05 +
    Math.min(quantity * itemPrice * 0.1, 100.0);
}
```

Split the expression into parts that are self-explanatory.

```java
double price() {
  double basePrice = quantity * itemPrice;
  double quantityDiscount =
    Math.max(0, quantity - 500) * itemPrice * 0.05;
  double shipping = Math.min(basePrice * 0.1, 100.0);

  return basePrice - quantityDiscount + shipping;
}
```
If the extracted variables can be used elsewhere in the code, you should use **extract method** instead.

### Introduce assertion

For the code to work correctly there are some conditions to be satisfied.

```java
// value must not be negative
public double squareRoot(double value) {
  // ...
}
```

Replace these commented assumptions with assertions.

```java
public double squareRoot(double value) {
  Assert.isTrue(value > 0);
  // ...
}
```

## Benefits

At first the benefits of removing comments might not worth the while. However, the payoff of refactoring is that:

- The code becomes more intuitive and obvious with self-describing names.
- Self documenting code keeps the documentation up to date. Separate comments are almost always guaranteed to fall back.
- Complex solutions are broken down to simpler ones.
- Instead of making assumptions, you are making programming errors visible through assertions.

{{< tweet user="nzkoz" id="538892801941848064" >}}

## Exceptions

Code comments are not always a bad smell. Sometimes comments can be useful.

- Instead of explaining what the implementation does, it is explained why something is implemented in a particular way.
- Even after simplifying a solution, a complex algorithm might need explanation.
- Writing Javadoc for public APIs or utilities is recommended.

## Summary

Code comments are a code smell. They might be written with good intentions but might indicate that the implementation should be simplified.

Complex expressions should be replaced with well named subexpressions. Blocks of code needing explanation should be extracted to their own well named methods.

Sometimes you might need to explain why something is implemented the way it is. Also complex algorithms might need an explanation. Javadoc for public interfaces should not be ignored.
