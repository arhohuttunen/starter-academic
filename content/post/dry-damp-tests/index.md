---
title: DRY and DAMP in Tests
subtitle: Achieving two different aspects of maintainability at the same time
date: 2021-02-14
authors:
  - arhohuttunen
summary: What are the DRY and DAMP principles? Why do people consider DRY and DAMP in tests contradictory and how can we achieve both?
categories:
  - Testing
tags:
  - Maintainability
  - Readability
image:
  focal_point: bottom
  preview_only: true
---

{{< youtube cD4ThqhnAp0 >}}
<br/>

In this article, we will look at the DRY and DAMP principles in the context of testing. We will see why people consider these principles contradicting when writing tests and how you can achieve both.

## :scroll: The DRY and DAMP principles

DRY and DAMP are principles that target two different aspects of maintainability. Both aim at making the code easier to change.

### Don't Repeat Yourself

DRY stands for _Don't Repeat Yourself_ and means that any piece of system knowledge should have only one representation. People often take DRY to mean that you shouldn't duplicate code, but it extends to far more than just that.

Things that are not conceptually related should not be related. A change to one thing should not cause a change to another. You want to minimize coupling between unrelated things.

The DRY principle increases maintainability by isolating the risk of change to smaller pieces.

### Descriptive And Meaningful Phrases

DAMP stands for _Descriptive And Meaningful Phrases_ and promotes the readability of the code. Making the code more readable makes it easier to understand. When the code is easier to understand, it's easier to maintain it.

The DAMP principle increases maintainability by reducing the time needed to read and understand the code.

## :exclamation: Contradiction in tests

You might have heard people saying that duplication is more acceptable in tests. You might also have heard that tests should be DAMP and not DRY. What does this mean in practice?

Let's take a look at an example. Please keep in mind that the example is very simple, and the illustrated problems would be much worse with more code.

```java
class DuplicationExampleTest {
    @Test
    void stopTaskProgress() {
        Task task = new Task();
        task.setTitle("Do the laundry");
        task.setStatus(Task.Status.IN_PROGRESS);
        task.setAssigneeId(1L);

        task.stopProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.OPEN);
        assertThat(task.getAssigneeId()).isNull();
    }

    @Test
    void finishTask() {
        Task task = new Task();
        task.setTitle("Do the laundry");
        task.setStatus(Task.Status.IN_PROGRESS);
        task.setAssigneeId(1L);

        task.finish();

        assertThat(task.getStatus()).isEqualTo(Task.Status.CLOSED);
        assertThat(task.getAssigneeId()).isEqualTo(1L);
    }
}
```

Arguably we have quite a bit of duplication going on in the construction. Quite often, in an attempt to remove the duplication, you see the following kind of solution.

```java
class BadExampleTest {
    private static final String TASK_TITLE = "Do the laundry";
    private static final Task.Status TASK_STATUS = Task.Status.IN_PROGRESS;
    private static final Long TASK_ASSIGNEE = 1L;
    
    @Test
    void stopTaskProgress() {
        Task task = new Task();
        task.setTitle(TASK_TITLE);
        task.setStatus(TASK_STATUS);
        task.setAssigneeId(TASK_ASSIGNEE);

        task.stopProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.OPEN);
        assertThat(task.getAssigneeId()).isNull();
    }

    @Test
    void finishTask() {
        Task task = new Task();
        task.setTitle(TASK_TITLE);
        task.setStatus(TASK_STATUS);
        task.setAssigneeId(TASK_ASSIGNEE);

        task.finish();

        assertThat(task.getStatus()).isEqualTo(Task.Status.CLOSED);
        assertThat(task.getAssigneeId()).isEqualTo(TASK_ASSIGNEE);
    }
}
```

The problem with a solution like this is that you now have coupling between tests. Changing something in the setup will affect all the tests.

I believe that code like this is often the result of static analyzers complaining about duplication. Tools like SonarQube will even suggest introducing a constant for the string literal. 

So what would happen if we tried to remove the code duplication by moving it to another method? 

```java
public class WorseExampleTest {
    private Task task;

    @BeforeEach
    void setupTask() {
        task = new Task();
        task.setTitle("Do the laundry");
        task.setStatus(Task.Status.IN_PROGRESS);
        task.setAssigneeId(1L);
    }

    @Test
    void stopTaskProgress() {
        task.stopProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.OPEN);
        assertThat(task.getAssigneeId()).isNull();
    }

    @Test
    void finishTask() {
        task.finish();

        assertThat(task.getStatus()).isEqualTo(Task.Status.CLOSED);
        assertThat(task.getAssigneeId()).isEqualTo(1L);
    }
}
```

Removing duplication like this reduces the readability. Before the change, you had all the details to understand a test inside the test. Now the details are hidden in the setup method.

Also, variation in test data setup now becomes a problem. Let's say we have to add another test for starting the progress on an open task. A very naive implementation would modify the test data locally in the test.

```java
    @Test
    void startTaskProgress() {
        task.setStatus(Task.Status.OPEN);
        task.startProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.IN_PROGRESS);
        assertThat(task.getAssigneeId()).isEqualTo(1L);
    }
```

The change makes it even worse because now the setup has been divided into two places. The construction happens in the test lifecycle method, and we then partially set the data in the test method.

Looking at examples like this, it feels like increasing DRY will reduce DAMP and the other way around. From this perspective, it makes sense to favor DRY in production code and DAMP in test code.

## :handshake: Complementary instead of contradictory

Let's take a small step back and look at the definition of DRY again. What does it mean that any piece of system knowledge should have only one representation?

What exactly is system knowledge in the context of tests?

Tests usually follow the arrange, act, assert pattern. You might first construct some objects, interact with those objects, and check the results. **Your tests have knowledge about how to implement these steps**. According to DRY, such knowledge should only have one representation.

On the other hand, what does DAMP mean in the context of tests?

It merely means that you would like to see and understand at one glance what happens in the test. **The most important information is what each step does, not how it's implemented**. Descriptive can refer to the purpose and meaningful can refer to relevancy.

If we move any test steps somewhere else, we will not reuse this knowledge but remove it from the test. It still does the same thing, but the knowledge is now indirect.

Let's think about the previous `Task` construction example. When we move the construction to `@BeforeEach`, the construction knowledge is not available inside the test.

```java
public class WorseExampleTest {
    private Task task;

    @BeforeEach
    void setupTask() {
        task = new Task();
        task.setTitle("Do the laundry");
        task.setStatus(Task.Status.IN_PROGRESS);
        task.setAssigneeId(1L);
    }
    
    @Test
    void stopTaskProgress() {
        task.stopProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.OPEN);
        assertThat(task.getAssigneeId()).isNull();
    }
```

Thinking in terms of system knowledge, we would like to apply DRY to _how_ to implement something. Same way, we would like to use DAMP to describe _what_ steps to take.

The construction of an object in the above example is _what_. How to construct that object is _how_. We want to be expressive about the _what_ and remove duplication around the _how_.

Putting DRY and DAMP in this perspective, the two things are not contradictory but complementary.

## :hammer: Test Data Builders

The test data builder pattern allows tests to specify only those parts of the objects that need to vary and use sensible defaults for those not relevant to the test.

Let's take a look at an example.

```java
public class TaskBuilder {
    private String title = "Not relevant";
    private Task.Status status;
    private Long assigneeId;

    public static TaskBuilder builder() {
        return new TaskBuilder();
    }

    public static TaskBuilder inProgressTask(Long assigneeId) {
        return builder().inProgressStatus().withAssignee(assigneeId);
    }

    public TaskBuilder inProgressStatus() {
        this.status = Task.Status.IN_PROGRESS;
        return this;
    }

    public TaskBuilder withAssignee(Long assigneeId) {
        this.assigneeId = assigneeId;
        return this;
    }
    
    // ...

    public Task build() {
        Task task = new Task();
        task.setTitle(title);
        task.setStatus(status);
        task.setAssigneeId(assigneeId);
        return task;
    }
}
```

The builder knows how to construct objects. The knowledge has only one representation. This knowledge is available from the tests in a descriptive way.

Let's take a look at how to use a builder in a test.

```java
public class BetterExampleTest {
    @Test
    void stopTaskProgress() {
        Task task = TaskBuilder.inProgressTask(1L).build();

        task.stopProgress();

        assertThat(task.getStatus()).isEqualTo(Task.Status.OPEN);
        assertThat(task.getAssigneeId()).isNull();
    }
```

You could say that this is much more descriptive than the original version. By moving the object construction knowledge into a builder class, we have achieved both DRY and DAMP.

{{% callout note %}}
**Additional reading:**

:pencil2: [How to Create a Test Data Builder](/test-data-builders/)
{{% /callout %}}

## :heavy_check_mark: Custom assertions

In our example, there is still some duplication in the way we are asserting results.

One way to remove this duplication while adding readability at the same time is to write custom assertions. I'm using AssertJ here, but you could write something like Hamcrest matchers as well.

```java
public class TaskAssert extends AbstractAssert<TaskAssert, Task> {
    protected TaskAssert(Task task) {
        super(task, TaskAssert.class);
    }

    public static TaskAssert assertThat(Task task) {
        return new TaskAssert(task);
    }

    public TaskAssert isOpen() {
        Assertions.assertThat(actual.getStatus()).isEqualTo(Task.Status.OPEN);
        return this;
    }

    public TaskAssert isUnassigned() {
        Assertions.assertThat(actual.getAssigneeId()).isNull();
        return this;
    }

    // ...
}
```

Now we can use the custom assertions together with our test data builders in the test.

```java
public class BetterExampleTest {
    @Test
    void stopTaskProgress() {
        Task task = TaskBuilder.inProgressTask(1L).build();

        task.stopProgress();

        assertThat(task).isOpen().isUnassigned();
    }

    @Test
    void finishTask() {
        Task task = TaskBuilder.inProgressTask(1L).build();

        task.finish();

        assertThat(task).isClosed().isAssignedTo(1L);
    }

    @Test
    void startTaskProgress() {
        Task task = TaskBuilder.openTaskWithAssignee(1L).build();

        task.startProgress();

        assertThat(task).isInProgress().isAssignedTo(1L);
    }
}
```

The result is very readable, it's fast to understand, and there is close to no duplication at all. We now follow both the DRY and DAMP principles very well.

You could argue that there is still a small duplication in the first two tests because both the tasks have the same assignee. Looking back in our first example introducing a constant was not a good idea.

So, what should we do about it? One solution is to use different assignees for the tasks in the two tests.

## Summary

DRY and DAMP in tests are not contradictory, but they target two different aspects of maintainability. To achieve both, we should extract the steps about how to do something and name those descriptively.

A couple of patterns that help with both concerns are test data builders and custom assertions. These patterns remove duplication in construction and verification while providing better readability and expressiveness.

The example code for this article can be found in [GitHub](https://github.com/arhohuttunen/write-better-tests/tree/main/dry-and-damp).
