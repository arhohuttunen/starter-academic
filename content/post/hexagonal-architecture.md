---
title: Hexagonal Architecture Explained
subtitle: Separating Business Logic From Infrastructure With Ports and Adapters
date: 2023-01-17
authors:
  - arhohuttunen
summary: Hexagonal Architecture is a way to structure the application so that it can be developed and tested in isolation from external technologies.
categories:
  - Software Craft
image:
  filename: stock/abstract-hexagonal-background.jpg
---

Hexagonal architecture is an architectural pattern introduced by Alistair Cockburn and written on his [blog](https://alistair.cockburn.us/hexagonal-architecture/) in 2005. The main idea is to structure the application so that we can develop and test it in isolation from external tools and technologies.

This is how Cockburn himself describes the architecture in one sentence:

>Allow an application to equally be driven by users, programs, automated test or batch scripts, and to be developed and tested in isolation from its eventual run-time devices and databases.
>-- <cite>Alistair Cockburn, 2005</cite>

In this article, we will look at some problems faced with traditional software projects. We will then learn about hexagonal architecture and how it tries to address those problems. We will also look at some implementation details and options for testing.

## The Problems With the Traditional Approach

Before going into details about hexagonal architecture, let's look at some typical problems we might face with large-scale applications.

On the front-end, **business logic of the application ends up leaking into the user interface**. As a result, this logic is hard to test because it's coupled with the UI. The logic also becomes unusable in other use cases and it's hard to switch from human-driven use cases to programmatic ones.

On the back-end, **business logic ends up being coupled to a database or external libraries and services**. This again makes the logic hard to test because of the coupling. It also makes it harder to switch between technologies, or renew our technology stack.

To remediate the problem of business logic and technological details mixing up, people often introduce a layered architecture. By putting different concerns on their own layer, the promise is that we can keep them nicely separated.

{{< figure src="/post/hexagonal-architecture/layered-architecture.svg" caption="Layered architecture" >}}

From one layer, we only allow components to access other components on the same layer or below. In theory, this should protect us from different concerns mixing up. The problem is that there is no clear mechanism to detect violations of this promise, and over time, we usually end up in the same situation we tried to avoid.

With the data access layer at the bottom, the **database drives the design**. When we design our use cases, we are primarily supposed to model behavior, but persistence is about storing the state. Wouldn't it be better to begin with the business logic of the application?

These entities easily leak to the upper layers, which ends up **requiring changes to the business logic when we make changes to the persistence**. If we have to change how we persist things, why should that have to change our business logic?

The above is a very simplistic view and rarely stays like that. In reality, we need to communicate with external services or libraries, and it's not always clear where these things belong.

{{< figure src="/post/hexagonal-architecture/layered-architecture-complex.svg" caption="Layered architecture with more components" >}}

When we need to add new components, the architectural layers need updating. This is prone to shortcuts, and **technical details leak into the business logic**, e.g. by directly referencing 3rd party APIs.

These observations should give us motivation to look for alternatives. Maybe there are some better ways to frame our architecture?

## What Is Hexagonal Architecture?

As already mentioned, the main idea of hexagonal architecture is to **separate the business logic from the outside world**. All the business logic lives inside the application, while any external entities are located outside of the application. The inside of the application should be unaware of the outside.

The goal is that the application can be controlled equally by users, other programs, or tests. We should be able to develop and test the business logic in isolation from frameworks, databases, or external services.

### Ports and Adapters

To make the separation of business logic and the outside world happen, the application only communicates with the outside world through **ports**. These ports describe the **purpose of conversation** between the two sides. It is irrelevant for the application what technical details are behind these ports.

**Adapters** provide connection to the outside world. They **translate the signals of the outside world** to a form understood by the application. The adapters only communicate with the application through the ports.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-external-dependencies.svg" caption="Separation of business logic and infrastructure in Hexagonal Architecture" >}}

Any port could have multiple adapters behind it. The adapters are interchangeable on both sides without having to touch the business logic. This makes it easy to grow the solution to use new interfaces or technologies.

For example, in a coffee shop application, there could be a point of sale UI which handles taking orders for coffee. When the barista submits an order, a REST adapter takes the HTTP POST request and translates it to the form understood by a port. Calling the port triggers the business logic related to placing the order inside the application. The application itself doesn't know that it is being operated through a REST API.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-flow-of-control.svg" caption="Adapters translate signals of the outside world to the application" >}}

On the other side of the application, the application communicates with a port that allows persisting orders. If we wanted to use a relational database as the persistence solution, a database adapter would implement the connection to the database. The adapter takes the information coming from the port and translates it into SQL for storing the order in the database. The application itself is unaware of how this is implemented or what technologies the implementation uses.

{{% callout note %}}
Many articles that talk about Hexagonal Architecture mention layers. However, the original article says nothing about layers. There is only the inside and the outside of the application. Also, it says nothing about how the inside is implemented. Whether we define our own layers, organize components by feature, or apply DDD patterns - it's all up to us.
{{% /callout %}}

### Primary and Secondary Adapters

As we have seen, some adapters invoke use cases of the application, while some others react to actions triggered by the application. The adapters that **control** the application are called **primary or driving adapters**, usually drawn to the left side of the diagram. The adapters that are **controlled** by the application are called **secondary or driven adapters**, usually drawn to the right of the diagram.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-primary-and-secondary-adapters.svg" caption="Primary and secondary adapters with use cases on the application boundary" >}}

The distinction between primary and secondary is based on **who triggers the conversation**. This relates to the idea from use cases of primary actors and secondary actors.

A primary actor is an actor who performs one of the application's functions. This makes the ports of the application a natural fit for describing the **use cases** of the application. A secondary actor is someone who the application gets answers from or notifies. This leads to secondary ports having two rough categories: **repositories** and **recipients**.

We should write use cases at the **application boundary**. A use case should not contain any detailed knowledge of the technologies outside the application. Hexagonal architecture can encourage the preferred way of writing use cases.

{{% callout note %}}
A typical mistake is that we write the use cases with knowledge about particular technologies. Such use cases are not speaking business language, become coupled with technologies used and are harder to maintain.
{{% /callout %}}

## Implementation

So far, we have only stated that the technical details should stay outside the application. The communication between the adapters and the application should only happen through ports. Let's look at what this means in practice.

### Dependency Inversion

When we implement a primary adapter on the driver side, an adapter has to tell the application to do something. The **flow of control goes from the adapter to the application through ports**. The dependency between the adapter and application points inwards, making the application unaware of who is calling its use cases.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-primary-adapter.svg" caption="Implementing primary adapters" >}}

In our coffee shop example, the `OrderController` is an adapter who calls a use case defined by the `PlacingOrders` port. Inside the application, `CoffeeShop` is the class who implements the functionality described by the port. The application is unaware of who is calling its use cases.

When we implement a secondary adapter on the driven side, **the flow of control goes out from the application** because we have to let the database adapter know it should persist an order. However, our architectural principle says that the application should not be aware of the details of the outside world.

To achieve this, we have to apply the [dependency inversion principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle).

> High-level modules should not depend on low-level modules. Both should depend on abstractions (e.g. interfaces). Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.
> -- <cite>Robert C. Martin, 2003</cite>

In our case, this is a fancy way of saying the application should not directly depend on the database adapter. Instead, the application should use a port, and the adapter should then implement that port.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-secondary-adapter.svg" caption="Implementing secondary adapters" >}}

The `CoffeeShop` implementation should not depend on the `OrdersDatabaseAdapter` implementation directly, but it should use the `Orders` interface and let `OrdersDatabaseAdapter` implement that interface. This inverts the dependency and effectively reverses the relationship.

We can also say that the `CoffeeShop` has a configurable dependency on the `Orders` interface, which is implemented by the `OrdersDatabaseAdapter`. Similarly, the `OrderController` has a configurable dependency on the `Orders` interface, implemented by the `CoffeeShop`. To configure these dependencies we can use dependency injection as an implementation pattern.

### Mapping in Adapters

Adapters should translate the signals of the outside world to something that the application understands and vice versa. Practically, this means that the adapters should map any application models to an adapter model and the other way around.

In our example, to make a distinction between the outer and inner model, we can introduce an `OrderRequest` model which represents the data coming in to the adapter as a REST request. The `OrderController` becomes responsible for mapping the `OrderRequest` into an `Order` model that the application understands. 

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-primary-adapter-mapping.svg" caption="Mapping of models in primary adapters" >}}

Similarly, when the adapter needs to respond to the actor calling it, we could introduce an `OrderResponse` model and let the adapter map the `Order` model from the application into a response model.

This might sound like extra work. We could just return models from the application directly, but this poses a couple of problems.

First, if we need to e.g. format the data, we then need to put **technology specific knowledge inside the application**. This breaks the architectural principle that the application should not know about the details of the outside world. If some other adapter needs to use the same data, reusing the model might not be possible.

Second, we are making it **harder to refactor** inside the application, since our model is now exposed to the outside world. If someone relies on an API we expose, we would introduce breaking changes every time we refactor our model.

On the other side of the application in our example, we could introduce an `OrderEntity` model to describe the details needed to persist the data. The technology-specific `OrdersDatabaseAdapter` is now responsible for translating an `Order` model from the application to something that the persistence layer understands.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-secondary-adapter-mapping.svg" caption="Mapping of models in secondary adapters" >}}

Again, it can be tempting to use a single model for the database entities and the application, but it comes with a cost. We would need to put **technology specific details inside the application model**. Depending on your technology stack, this could mean that you now have to worry about details like transactions and lazy loading inside your business logic.

## Testing

One goal of hexagonal architecture mentioned was the ability to test the business logic in isolation from external tools and technologies. This is something that comes naturally from the separation of concerns implemented with ports and adapters. Without this separation, our options for testing are much more limited and there is a tendency for broader tests.

### Testing the Business Logic

The first step in implementing a use case would be to start with a test describing it. We begin with the application as a black-box and allow the test only to call the application through its ports. We should also replace any secondary adapters with mock adapters.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-unit-test.svg" caption="Unit testing the business logic" >}}

While it is possible to use a mocking framework here, writing your own mocks or stubs will prove valuable later. For any repository adapters, these mocks could be anything as simple as a map of values.

### Testing Primary Adapters

The next step is to connect some adapters to the application. We would typically start from the primary adapter side. This allows the application to be driven by some actual users.

We can keep using the mock adapters from the last step for the secondary adapters. Our narrow integration tests will then call the primary adapter to test it. In fact, we could ship a first version of our solution with the secondary adapters implemented as stubs.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-primary-adapter-integration-test.svg" caption="Testing the primary adapters" >}}

For example, our integration test could make some HTTP requests to the REST controller and assert that the response matches our expectations. Although the REST controller is calling the application, the application is not the subject under test.

If we use a test double for the application in these tests, we will have to focus more on verifying interactions between the adapter and the application. When we only mock the right side adapters, we can focus on state-based testing.

{{% callout note %}}
We should make sure to only test the responsibilities of the controller in these tests. We can test the use cases of the application on their own.
{{% /callout %}}

### Testing Secondary Adapters

When it's time to implement the right side adapters, we want to test that the integration to the 3rd party technology works correctly. Instead of connecting to a remote database or service, we can containerize the database or service and configure the subject under test to connect to that. 

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-secondary-adapter-integration-test.svg" caption="Testing the secondary adapters" >}}

For example, in the Java world, it's possible to use something like Testcontainers or MockWebServer to replace the real remote database or service. This allows us to use the underlying technology locally without having to rely on the availability of external services.

### End-To-End Tests

Although we can cover the different parts of the system with unit and integration tests, it's not enough to weed out all problems. This is when end-to-end tests (also known as broad integration tests or system tests) come in handy.

{{< figure src="/post/hexagonal-architecture/hexagonal-architecture-end-to-end-test.svg" caption="End-to-end testing the system" >}}

We can still isolate the system from external services, but test the system as a whole. These end-to-end tests execute entire slices of the system from primary adapters to the application to the secondary adapters.

What we are looking for in these tests is executing the main paths of the application. The intent is not to verify the functional use cases, but that we wired the application together correctly and it is working.

{{% callout note %}}
This approach will evidently lead to some overlapping tests. To avoid testing the same things on different levels repeatedly, it's important to think about the responsibilities of the subject under test.
{{% /callout %}}

## Advantages and Disadvantages

Good architecture allows the software to be constantly changed with as little effort as possible. The goal is to minimize the lifetime costs of the system and maximize productivity.

Hexagonal architecture has several advantages that fulfill these premises:

- We can delay decisions about details (such as what frameworks or database to use).
- We can change the business logic without having to touch the adapters.
- We can replace or upgrade the infrastructure code without having to touch the business logic.
- We can be promote the idea of writing use cases without technical details.
- By giving explicit names to ports and adapters, we can better separate concerns, and reduce the risk of technical details leaking into the business logic.
- We get options on testing the parts of the system in isolation as well as grouped together.

As with any solution, hexagonal architecture has its disadvantages.

- Can be over-engineering for simple solutions (such as CRUD applications or a technical microservice).
- Requires effort on creating separate models and mapping between them.

In the end, the decision to use hexagonal architecture comes down to the complexity of the solution. It's always possible to start with a simpler approach and evolve the architecture when the need arises.

## Summary

The main idea of hexagonal architecture is to separate business logic from the technical details. This is done by isolating these concerns with interfaces.

On one side of the application, we create adapters that use the application interfaces. These can be, for example, controllers that drive the application. On the other side of the application, we create adapters that implement the application interfaces. These can be, for example, repositories that the application gets answers from.

In the next article, we will look at how to implement hexagonal architecture in a Spring Boot application.