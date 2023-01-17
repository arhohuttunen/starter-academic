---
title: Code Quality Checks
date: 2016-06-13
authors:
  - arhohuttunen
categories:
  - Software Craft
image:
  filename: stock/detective-smoking-pipe.jpg
---

Code quality is a weak spot in nearly every software project. This is especially true for legacy projects. What once was elegant, over time became rougher and finally incomprehensible.

Monitoring and fixing code quality issues is something that has been **proven to increase the quality of the application and decrease the delivery time to stakeholders**. Unfortunately, a large portion of developers do not monitor for code quality or ignore fixing any quality issues.

The first step in adding code quality checks to your code is to know the available tools. The second step is to know how to use these tools and when.

## Choosing your weapon

Addressing code quality checks can be roughly divided into three categories: code reviews, static analysis and dynamic analysis. Each category addresses issues that are hard to discover by another category.

### Code reviews

Manual code reviews should focus on the things that automation is not that good looking at. This includes code smells, application design issues or potential problems in a solution.

**Pair programming** is the fastest way to get a second opinion about your code. It's an informal way of conducting a code review at the very moment that you are developing the code.

You can make **peer reviews** by asking someone to look at your code. If the version control tools permit you can also make a **pull request**. You need to keep the granularity of peer reviews small enough. Otherwise the reviewer will feel overwhelmed and ignore most of the content.

{{< tweet user="iamdevloper" id="397664295875805184" >}}

Formal code inspections are a breeze from the past and I do no recommend them. The scope of the review is usually too large and the time of the review is usually too late.

### Static code analysis

You can perform static code analysis on the source code without actually executing it. It's a great way to find bugs, drive for better practices or to enforce a coding convention.

These are things that machines are better finding at than humans, so it makes sense to use these tools for this purpose. Using them allows manual code reviews to focus on larger things.

The most commonly used static analysis tools for Java development in order of popularity are [SonarQube](http://www.sonarqube.org), [SpotBugs](https://spotbugs.github.io), [Checkstyle](http://checkstyle.sourceforge.net) and [PMD](https://pmd.github.io). Each of the tools address different issues and are best used together. SonarQube can incorporate the results of the others in its reports.

### Dynamic code analysis

The difference between static and dynamic code analysis is that you need to run the latter during the execution of the application.

When you notice that your application is slow you can fire up **profilers**. You can figure out where the time is spent much easier if you use a profiler than if you just look at the source code.

**Memory tools** can either show some statistic in real-time or analyze some traces created during the execution. You can run these tools when you encounter performance problems or out-of-memory errors.

You can use **monitoring tools** to check things like number of database transactions or the actual database queries. These tools are often the final guards reporting about near failure before a service dies because of some resource shortage.

## Shortening the feedback loop

Fixing any quality issues is cheaper the sooner you do it. While monitoring for quality issues is great it does not force you to fix them.

If you postpone making fixes it can result in the unwanted effect of getting overwhelmed by the number of issues. This further discourages fixing any issues and the vicious circle is ready.

**Pair programming and peer reviewing are effective ways of getting quick feedback about the code**. It usually also has the psychological effect of making people try their best because they know that someone will immediately see their piece of work.

**SonarQube is great for reporting and getting statistics over time**. However in my experience, it is not so great for enforcing coding standards. The feedback loop from committing a change to receiving the results of the analysis is simply too long. It takes too much discipline to fix anything afterwards.

**The shortest static analysis feedback loop is achieved by making code quality checks while you are writing the code**. You can do this in your IDE using plugins for tools like [SonarLint](https://www.sonarlint.org) or [ESLint](https://eslint.org/). The great thing about this approach is that you get very fast feedback about any issues and can address them at once.

## Preventing rot

**To enforce any rules you should make your continuous integration build break in case of violations**. This means you should add static analysis tools like PMD, SpotBugs and Checkstyle as parts of your build.

This might sound crazy at first but the best way to prevent software rot is, well, to prevent it. It works very well in green field projects but in legacy projects you cannot be so absolute. To manage this you can adjust the rulesets that you use and work gradually from there.

You can achieve best results by adding all levels of code quality checks into your development process. This makes sure that you get very fast feedback about any issues, still get notified by the continuous integration build if you missed something and get decent reporting for the analysis.

{{% callout note %}}
**Additional reading:**

- [How to prevent legacy code from emerging](/prevent-legacy-code-from-emerging)
- [Strict Control of Java Code Quality](https://www.yegor256.com/2014/08/13/strict-code-quality-control.html) by Yegor Bugayenko
- [Developer Productivity Report 2013 – How Engineering Tools & Practices Impact Software Quality & Delivery](https://jrebel.com/rebellabs/developer-productivity-report-2013-how-engineering-tools-practices-impact-software-quality-delivery/) by RebelLabs
{{% /callout %}}

In following posts I will add static analysis tools into Maven and Gradle builds, and discuss what to look for in code reviews.
