---
title: "Four No Bullshit Ways to Easily Increase Your Programming Productivity"
date: 2021-01-17
authors:
  - arhohuttunen
summary: Deepen your skills, automate repetitive tasks, shorten your feedback loops, and code less to achieve greater productivity.
categories:
  - Productivity
image:
  caption: Photo by [Matthew Fournier](https://unsplash.com/@mattyfours) on Unsplash
  filename: stock/black-laptop-displaying-source-code.jpg
---

Programming can be quite repetitive at times. We end up typing the same commands over and over, or we copy paste-stuff from one place to another.

However, the machines we are using are pretty good at doing such tedious, repetitive tasks. Utilizing this ability is surprisingly underrated, and a lot of people keep doing what they do out of habit.

> Most people spend more time and energy going around problems than in trying to solve them.
> -- <cite>Henry Ford</cite>

In this post, we will look at four ways to increase programming productivity. I am mentioning some tools, but these tips are by no means tool-specific. The presented principles are more important. 

## :hammer_and_wrench: Learn the tricks of your IDE

IDEs are more than just a code editor. They try to bring together all the tools you need to compile, debugging, and much more. 

To get the most of your IDE, **learn all the shortcuts** of actions you use most. Learning the shortcuts can easily save you time browsing the code, writing, editing, running, or debugging. Every time you need to look up a command using your mouse, try to memorize the shortcut.

IDEs come with **tools and plugins** that allow easy integration to other tools and services. With the right set of tools, you don't have to leave your IDE to do separate tasks. Instead, you can do the tasks from the IDE.

**Learn how to refactor using your IDE**. Nowadays, the tools available for this are excellent, and the IDE does a more robust job than you ever will. Do you need to rename a method? Just let the IDE do it for you, and it will rename all the method calls as well. 

It would be best if you also learned how to **use snippets or live templates**. These allow you to insert common constructs into your code, like loops, conditions, or more complex declarations. It is like auto-completion on steroids. IntelliJ IDEA even has a [productivity guide](https://www.jetbrains.com/help/idea/productivity-guide.html) that shows how much it has saved you from typing.

What IDE you use is not that important; what is important is that you maximize that IDE's potential.  

## :mechanical_arm: Automate repetitive tasks

If you find yourself doing something over and over again, you should consider automating that. Even if these are not programming-related tasks, you already possess the skills to make the computer do the work for you as a programmer.

**Utilize desktop automation tools** available for your platform. For Linux, there is [AutoKey](https://github.com/autokey/autokey). For Mac, you have [Automator](https://support.apple.com/en-gb/guide/automator/welcome/mac), and for Windows, there is [AutoHotKey](https://www.autohotkey.com/). There are many, many more, but these are just a couple of examples.

**Learn to use scripting tools**. Learning a little bit of shell scripting or Python can go a long way. Sometimes the editing or searching capabilities of a text editor are simply not enough. With scripting skills, you can easily manipulate text or search something from log files, for example.

**Spend more time on the command line**. There is a ton of stuff that you can do on the command line. Let's say you need to edit a bunch of images the same way. You could do it in the image editor or try to use something like [ImageMagick](https://imagemagick.org/) instead. If this is something that you need to do often, automating it could save a lot of time.

Also, when working on the command line, you might find yourself typing the same commands regularly. You can **write aliases for your most common commands**.

And while you are on the command line, why not use something like Z Shell and [Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh) that give you [features](https://github.com/hmml/awesome-zsh) like git auto-completion and much more. For Windows users, you could alternatively try [posh-git](https://github.com/dahlbyk/posh-git).

## :leftwards_arrow_with_hook: Shorten your feedback loops

The common thing with all feedback loops is that as the time waiting goes up, so does the cost to fix something. As the feedback gets slower, there is a higher chance you have introduced changes that are already in conflict.

I have written about [better git commits](/make-better-git-commits). **Committing more often makes it easier to go back to a working state** when making mistakes. Committing more often also makes the commits more focused and separates logical changes. 

**Pushing to a remote repository more often reduces the risk of merge conflicts**. It's not completely removing them but makes it less frequent and easier to deal with them. Another obvious benefit of pushing more often is that you will have a second copy of your work.

Working with other people, you might be doing code reviews. You make a pull request, someone checks it when he has time, and you have to back and forth while switching tasks in between. Does this sound familiar?  

I call this pull request ping pong, and it can be a colossal waste of time. You might want to **try pair-programming**. And by trying, I mean don't just try it once and say you didn't like it. Experiment with it for a month and see if it helps.

{{< tweet user="iamdevloper" id="397664295875805184" >}}

The small tasks that you do tens or hundreds of times a day are called micro-feedback loops. You can **save a ton of time optimizing your micro-feedback loops**. Making a build and tests take 30 seconds instead of 2 minutes doesn't sound much, but it adds up. That could be easily tens of minutes a day or hundreds of minutes a week. 

Also, **small pauses can be long enough to get you distracted**, decide to open the email or switch context while waiting. Research shows that it can take up to 23 minutes to get back to the state of flow.

When the feedback loops are too long, you might run them less often and start batching things. Lower frequency, in turn, makes the feedback loop even longer.

Incorporating a code formatter to be run on saving, adding a static analysis plugin to your IDE, or adding a git hook that runs the tests can be effective ways of **making sure your feedback loops are frequent**. I have also written before about [code quality checks](/code-quality-checks) and shortening the feedback loop.

## :stop_sign: Write less code

Being aware **less code is better** is probably the step where you can save the most time. We have the best of intentions to do a good job, which can also turn against us.

> It's painful for most software developers to acknowledge this, because they love code so much, but the best code is no code at all.
> -- <cite>Jeff Atwood, Stack Overflow co-founder</cite>

Every time we start writing new code, we should **think about whether something is required at all**. Can you get away with it without writing it? Is this feature truly needed?

It's easy to get into the trap of adding functionalities or features that you think might be useful. But, quite frankly, every functionality should have an actual use case before implementing it. 

The same goes for future-proofing something. Just because you _think_ something might be useful later doesn't mean you have to implement it right now. The YAGNI principle applies here: _**You Aren't Gonna Need It**_.

> Always implement things when you actually need them, never when you just foresee that you need them.
> -- <cite>Ron Jeffries, Extreme Programming co-founder</cite>

**Premature use of design patterns** can result in over-engineering. Don't get me wrong; sometimes, you might know beforehand that you need to use a specific pattern. However, there are times when you need to recognize multiple occurrences of something in the code before you can decide. Deferring decisions can save a lot of time. 

Finally, **premature optimization** can be a huge waste of time. It's no different with optimizing something that has minimal effect. The Pareto principle states that for many outcomes roughly 80% of consequences come from 20% of the causes.

These things address different dimensions of the code. They have in common, though, that they all affect the dimension of **time spent coding**. The key here is to be conscious of these trade-offs.

## Summary

Knowing the tools that we use very well makes us more effective. Learning them allows us to use them without thinking. 

Computers are much better at doing repetitive tasks than humans. As programmers, we can make the computer work for us by automating tasks.

Fixing issues is more manageable the sooner you find them. Shortening your feedback loops guarantees that you don't spend extra time on rework.

The best code is no code at all. We should pay attention not to not implement more than is needed.
