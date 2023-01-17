---
title: Make Better Git Commits
date: 2016-01-06
authors:
  - arhohuttunen
categories:
  - Software Craft
image:
  filename: stock/personal-computer-showing-development-process.jpg
---

Version control systems provide a way to track changes between source code versions. Git is most likely the most commonly used one nowadays. Git can be used in different ways and there is no one correct way to do it. However, there are some pitfalls that might make tracking of changes harder than it should be.

There are a lot of more thorough articles like [Commit Often, Perfect Later, Publish Once](https://sethrobertson.github.io/GitBestPractices/) on the subject. Here is my take on the matter. Rather than just saying don't do this and don't do that I'll try to show some best practices that can make your life easier. The guidelines here are not necessarily Git specific but could be applied to some other version control systems as well.

## Commit often

When you are committing to a local repository it doesn't really matter how often you commit. Or rather, the **more often you make commits the better**. You can commit to your heart's content.

When you commit often enough **it is always a lot easier to go back when you make a mistake**. It's a lot easier to revert to a commit rather than try to undo your changes by yourself.

Sometimes you need to do some experimental commits because you are not sure what you are ending up with. You maybe need to add something to the latest commit because you forgot something. In this case you can [use the commit amend option](https://www.atlassian.com/git/tutorials/rewriting-history/git-commit--amend) to add something to the latest commit.

If you don't want to include the whole commit history when pushing your changes to remote repository you can also [use rebase interactively](https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase-i) to change the history. This should be only used to clean up a mess and not to combine all the changes as a single commit.

## Single logical change at once

If by the time you are starting to commit you notice that you need to list the changes you have made you have already done too much at once. **Try to separate logical changes and don't do too much at once.** This makes it a lot easier to track changes.

Imagine that you have a larger commit with several files in it. The commit message states that you have done five separate things. Now you need to know where one of those particular changes has been done. This will be difficult.

If the aforementioned five separate things are done in separate commits tracking the changes quickly becomes easy. Each commit now has less changed files and the commit message tells exactly what has been changed.

You can also **stage your changes in hunks** by the time you start committing. This can be done if you notice that you have done too much at once. Simply [select parts of a file in a separate commit](https://git-scm.com/book/en/v2/Git-Tools-Interactive-Staging).

Tools like SourceTree among others provide a graphical user interface alternative to interactive staging. There is a **Stage hunk** option which only stages part of a file for the commit.

## Separate formatting

If you decide to improve the formatting of the code always **do formatting in a separate commit**. Formatting changes along with code changes will make it a lot harder to track changes.

When you make changes to the code make sure you don't mess with the formatting so that **diff tools can show the exact change**. Then fix the formatting separately.

## Tell what you did and why

Writing meaningful commit messages is crucial. Anyone who is trying to find out why something has changed in the code is going to read the commit messages.

A good rule of thumb is to **start with a separate title line** which shortly describes what has changed. Then insert a couple of line breaks and **write a longer explanation about what and why**.

Another good rule of thumb is to always **include a issue tracking system ticket ID** in the commit message. If you are using JIRA for example you should always include the JIRA issue ID. This allows tracking between commits and the issue tracking system.

## Summary

Here some basic guidelines for better Git commits were presented. Basically you need to remember to:

- Commit often
- Commit single logical change at a time
- Keep formatting changes separate from code commits
- Write meaningful commit messages

Can you think of something else? What would you suggest to make better Git commits?
