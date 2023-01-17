---
title: Avoiding Unnecessary Null Checks
date: 2015-09-21
authors:
  - arhohuttunen
categories:
  - Software Craft
image:
  filename: stock/headless-pensive-businessman.jpg
---

The most common reason for writing null checks is that you run into a null pointer exception. The second most common reason is that you happened to think about it at some certain case.

The problem is that you are not probably handling null in every single method call. This means that there are potential bugs lurking everywhere.

Null pointer exceptions are bad. Would it not be better if you did not have to check for nulls at all?

> I call it my <a href="https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions">billion-dollar mistake</a>. It was the invention of the null reference in 1965. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.
> -- <cite>Sir Tony Hoare, 2009</cite>

## Try not to return null

Generally speaking, returning null from a method should be considered really bad. This forces the user of the method to do null checks and create conditional code paths.

There are three ways of avoiding returning nulls:

- Returning null objects.
- Throwing exceptions.
- Move behavior into an object to go with the data.

### Returning null objects

One way of avoiding returning null is using the [Null Object pattern](https://en.wikipedia.org/wiki/Null_Object_pattern). Basically you return a special case object that implements the expected interface. Instead of returning null you can implement some kind of default behavior for the object. Returning a null object can be considered as returning a neutral value.

Using a null object:

```java
public class User {
  private boolean authenticated;

  public boolean isAuthenticated() {
    return authenticated;
  }
}

public class NullUser extends User {
  @Override
  public boolean isAuthenticated() {
    return false;
  }
}

// Somewhere initialize user as NullUser at declaration
// ...
User user = getCurrentUser();
if (!user.isAuthenticated())
  redirectToUnauthorizedPage();
```

An empty collection is another example of the null object pattern. Returning an empty collection is always better than returning null. You can safely iterate an empty collection.

Returning null:

```java
Collection<Item> items = getItems();
if (items != null) {
  for (Item : items) {
    item.doSomething();
  }
}
```

Returning an empty collection instead:

```java
Collection<Item> items = getItems();
for (Item : items) {
  item.doSomething();
}
```

### Throwing exceptions

When appropriate, you should fail fast by throwing an exception instead of polluting the caller code with multiple null checks. If the data passed in is not sufficient complain by throwing an exception.

Also, you should not hide exceptional situations by catching exceptions and returning nulls. In object-oriented programming exceptions are supposed to be let bubble up.

### Move behavior into an object to go with the data

Most articles I have read about avoiding null talk about null objects or throwing exceptions. There is however a third way of avoiding nulls that addresses improving the design. Following the _Tell, Don't Ask principle_ it is possible to change the way non-existing information is handled.

Consider an example where it is possible to send customer service announcements to a customer to a postal address or via email. A customer could have given either a postal address or an email address.

A naive approach might query the customer object for these attributes, do null checks and then use the appropriate method to send an announcement.

Conditional logic:

```java
public class Customer {
  private PostalAddress address;
  private EmailAddress emailAddress;
}

// Somewhere in the code
if (customer.getPostalAddress() != null) {
  sendPostTo(customer.getPostalAddress());
} else if (customer.getEmailAddress() != null) {
  sendEmailTo(customer.getEmailAddress();
}
```

A better way is to provide interfaces for customer service announcement and a communication method that make it possible to hide the details within the customer class.

```java
public interface CustomerServiceAnnouncement {
  void sendByEmail(EmailAddress address);
  void sendByPost(PostalAddress address);
}

public interface CommunicationMethod {
  void send(CustomerServiceAnnouncement announcement);
}

public class EmailAddress implements CommunicationMethod {
  public void send(CustomerServiceAnnouncement announcement) {
    announcement.sendByEmail(this);
  }
}

public class Customer {
  private PostalAddress address;
  private EmailAddress emailAddress;
  private CommunicationMethod preferredCommunication;
  ...
  public void tell(CustomerServiceAnnouncement announcement) {
    preferredCommunication.send(announcement);
  }
}
```

Not only this removes the need for null checks but also breaks up the conditional logic into composition of objects.

{{% callout note %}}
**Additional reading:**

- [Switch statements](/switch-statements/)
{{% /callout %}}

## Never pass null

For code that you own you should get rid of passing nulls. This way you do not even need to guard against them.

There are several ways of making sure that your parameters are never null:

- Always initializing variables when they are introduced.
- Using the [Builder pattern](/test-data-builders) to completely construct objects before being created.
- Providing default values for properties or lazy loading.

## There is grey to every story

As always everything is not so black and white. Sometimes you do not own the code or are using third party libraries that are returning null. In these cases you do not have any option but check for nulls.

Another case is when you are writing an external API that exposes some methods. You probably want to check the passed in parameters and throw some exceptions letting the user know about invalid input.

Sometimes null might be a valid functional result, e.g. if you are trying to find a single element from database matching certain query parameters. In such case null value indicates that no such entity was found.

You might still argue that you need to guard against developers passing in null. While it is partly true, it is better to focus your efforts in-house in never passing in null instead of null check. By doing so you not only reduce the number of lines of code written but also are forced to use better design practices.

{{% callout note %}}
**Additional reading:**

- [Why NULL is Bad?](https://www.yegor256.com/2014/05/13/why-null-is-bad.html) by Yegor Bugayenko
- [Say "No" to "Null"](https://elegantcode.com/2010/05/01/say-no-to-null/) by John Sonmez
- [Avoiding Nulls with Polymorphic Dispatch](http://www.natpryce.com/articles/000778.html) by Nat Price
- [Avoiding Nulls with "Tell, Don't Ask" Style](http://www.natpryce.com/articles/000777.html) by Nat Price
{{% /callout %}}
