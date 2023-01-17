---
title: How Do I Test Private Methods?
date: 2021-02-07
authors:
  - arhohuttunen
summary: What is the best way to test private methods? Should I use some library or use reflection? Can I just make the method public?
categories:
  - Software Craft
tags:
  - testing
image:
  focal_point: topleft
  filename: stock/man-using-laptop.jpg
---

> What is the best way to test private methods? Should I use some library or use reflection?

Don't test private methods directly. The best way to test a private method is via another public method. Private methods are an implementation detail of a public method.

> But, but... it might be tough to test the private method through the public method. 

If that is the case, then there is a design smell near the class that you are testing. If the test setup is too complicated, the class might try to do too much. You should take a look at if you can extract some functionality in a new class.

> But, but... if you add more classes, isn't it adding complexity?

In most cases, adding a new class reduces complexity instead of increasing it. When a class has a clear, small responsibility, it's easy to understand and test.

> But, but... so you are saying write 10 small classes instead of private methods just to test it?

I'm not suggesting that you should create a new class just to test a method. I'm suggesting that if you listen to your tests, you can improve your design. Things like a complex test setup are clues about the implementation complexity.

> But, but... can't I just make the method public?

It's not a good idea to expose a method just for testing. It will break encapsulation and leak the internals for everyone to use, which can have harmful side effects. The urge to test a private method might mean that it's part of separate responsibility that belongs to another class.

> But, but... if I can't make the methods public, isn't extracting to a separate class doing the same thing?

If moving something to another class bothers you, there are more visibility modifiers than just `public` and `private`. There's also `protected` and `package-private`, and you can use this to your advantage. If your test class is located in the same package as the target class, there is no problem.

> But, but... I want to test the code in isolation, aren't unit tests supposed to be isolated?

Isolation usually means isolation from external dependencies like services, databases, or the file system.

If you test private methods directly, your tests will break every time you refactor your class. On the other hand, if you only test public methods, you can refactor the class internals without breaking the tests.

> But, but... aren't the tests and the code supposed to go hand in hand?

Yes, but only on the public contract level. If you directly test the implementation and someone wants to make a small change to it, he will have to change the tests too.

If you do this repeatedly, unit tests start to hinder development and become an annoyance to the development team. If you cannot change your implementation without changing the tests, your test strategy is lacking.

> But, but... what if I like to take a bottom-up approach and don't have a public method when I write the private methods?

Maybe you should try a top-down approach. If you write the private methods first, you are guessing what you need. If you end up not being happy with the solution, you will have to rewrite all the tests. Starting with the public method makes you think about how it's going to be used.

> But, but... if I develop the solution and the tests don't pass, isn't it hard to know where the problem is?

Not really. You wouldn't implement everything at one go. You would start with a simple test and a simple solution.
Now you can piece by piece add tests and functionality. You get the same coverage and can refactor the implementation without breaking your tests.

> Hmm. Well, I'm not 100% convinced yet, but you have made some good points. Let me think about this.

Great. Let me know what do you think a little later.
