---
title: How to Prevent Legacy Code From Emerging
date: 2015-05-22
authors:
  - arhohuttunen
categories:
  - Programming
tags:
  - Refactoring
  - Clean code
image:
  filename: stock/broken-windows-in-an-abandoned-house.jpg
---

Every software developer has faced the situation. You have been assigned a task to add or change a feature. You know nothing about the particular feature but it does not sound too complex.

This is what you think until you look at the code. You quickly realize that the code is a mess. A monolithic monster with a lot of copy-paste code, no comments, implementation full of anti-patterns and there are no tests at all.

You look at the commit log and curse the person who wrote this big pile of mud. You feel frustrated because it is very hard to get a grasp of what is going on in the code and making changes seems impossible as you are afraid to break something.

## How does legacy code emerge?

You find out promptly that this code needs a major refactoring. The problem is however that the deadline is approaching and you simply have no time. Refactoring is not accounted for in the estimates. You also feel that the original author should be responsible for the refactoring.

Here is the unpleasant breaking news: **the legacy code is starting to emerge because of your own actions**.

Let that sentence sink in for a moment.

There is noone else to blame. You let the code rot start. You made it easy for the next developer to skip unit tests because you did not add them in the first place. This is called the [Broken windows theory](https://en.wikipedia.org/wiki/Broken_windows_theory).

Every piece of code written and managed by several people will eventually turn into legacy code unless you conduct conscious actions to prevent and fix it. Legacy code will emerge even if you do not intentionally do it. This is because adding features that were not natural part of the original implementation will slowly turn into spaghetti code.

> As an evolving program is continually changed, its complexity, reflecting deteriorating structure, increases unless work is done to maintain or reduce it.
<cite>Meir Manny Lehman, 1980</cite>

Anyone can create clean code when starting from scratch. Keeping the code clean for a longer period requires conscious maintenance.

## How do I prevent legacy code from emerging?

The first and foremost thing to do is to apply the **Boy Scout Rule**: _always leave the code behind in a better state than you found it_. You will need to develop a discipline to do this. You should not need a motivational speech to do it.

The second thing is to realize that refactoring is not some external activity that is performed on request. **Refactoring should be an ongoing process that is part of your daily work**. When you have been assigned to add a feature your mission is to do preparatory refactoring so that adding the feature is natural.

The third thing is tests. Tests are important part of the maintenance process because they will remove the fear of changing things. **Tests are your safety net for not creating unintentional changes**.

> To me, legacy code is simply code without tests.
<cite>Michael C. Feathers</cite>

Legacy code doesn't magically appear. Do not be that guy. Do not take shortcuts and contribute to the birth of legacy code.
