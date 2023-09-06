{
:title "Objects, Values and API Design"
:tags ["Design" "Java"]
:toc true
:description "Some OO languages like Java have historically only provided one way to create new types: the <code>class</code> keyword. But there's a difference between defining value types and object types, and that difference bleeds over in to API design."
}

## Objects and Values: What's the Difference?
For the purposes of this discussion, when I talk about *objects*, I'm talking about little bundles of data that have associated **behaviour**. This is different from what I will call *values*, which are simply little bundles of data, no behaviour.

For example, I may have an object whose job it is to calculate bounding boxes around some shapes, and apply some specified amount of buffer space or padding. The class for this object may look something like this:
```java
public class BoundingBoxCalculator {

  private final int padding;

  public BoundingBoxCalculator(int padding) {
    this.padding = padding;
  }

  public Shape calculateBoundingBox(List<Shape> shapes) {
    // Perform the calculation and apply the padding
    return boundingBox;
  }
}
```
This is in contrast to a value, which does not have any behaviour: it's simply a container to hold related data together. The standard, braindead, obvious approach to writing one of these is as follows:
```java
public class UserAndDateTime {

  private String username;
  private LocalDateTime datetime;

  public UserAndDateTime(String username, LocalDateTime datetime) {
    this.username = username;
    this.datetime = datetime;
  }

  public String getUsername() {
    return username;
  }

  public void setUsername(String username) {
    this.username = username;
  }

  public LocalDateTime getDateTime() {
    return datetime;
  }

  public void setDateTime(LocalDateTime datetime) {
    this.datetime = datetime;
  }

  @Override
  public boolean equals( Object o ) {
    if ( this == o ) return true;
    if ( o == null || getClass() != o.getClass() ) return false;
    UserAndDateTime that = (UserAndDateTime) o;
    return Objects.equals(username, that.username) &&
          Objects.equals(datetime, that.datetime);
  }

  @Override
  public int hashCode() {
    return Objects.hash(username, datetime);
  }
}
```
We'll see a bit later that this is an awful implementation, and we can do significantly better. But first, let's take another look at our `BoundingBoxCalculator`.

## A(n Opinionated) Lesson in API Design
So, we have a `BoundingBoxCalculator`. How do we use it? Well, it's a Java object like any other, so let's create one and put it to work!
```java
List<Shape> shapes = ...
BoundingBoxCalculator calculator = new BoundingBoxCalculator(5);
Shape boundingBox = calculator.calculateBoundingBox(shapes);
```
Okay, that works, and for a lot of code we write, it's good enough: it compiles fine, it does it's thing, and it makes sense if you know how to read code. But it's ugly, and it's certainly not **fun**. It's got that magic number 5 passed to the constructor, it's verbose, and it *reads like code*. Sure, we can look at the constructor's method signature to see what that number does, but that's one step removed from *this* code. We can also get rid of that ugly magic number by introducing a variable, but now the code is just more verbose and even *less* fun to read:
```java
List<Shape> shapes = ...
final int padding = 5;
BoundingBoxCalculator calculator = new BoundingBoxCalculator(padding);
Shape boundingBox = calculator.calculateBoundingBox(shapes);
```
The goal of good code is to *communicate with other programmers*. We shouldn't be writing code for the computer, we should be writing code for each other! Wouldn't it be nicer to say:
```java
List<Shape> shapes = ...
Shape boundingBox = BoundingBoxCalculator.withPaddingOf(5)
                                         .calculateBoxAround(shapes);
```
We've accomplished exactly the same thing here, but the code reads more like natural language; the intent of the code is clear not only to the compiler, but also to our fellow programmers (and our future selves!).
## What About Values? Can They Be Pretty Too?
I mentioned earlier that the "getters and setters" version of the `UserAndDateTime` value is an awful implementation. It's awful for (at least) two reasons:
1. Instances of `UserAndDateTime` can be **mutated**. This does not make for a very good value: it can change at any time! Values are, by definition, set in stone.
2. The API is awfully verbose. It makes sense from a JavaBeans perspective, but not from the perspective of a human reader. JavaBeans were designed with tools in mind, not programmers; tools need an obvious convention to follow if they are going to extract information dynamically.

> On the subject of JavaBeans (or their watered down cousins, POJOs), its interesting to consider how Groovy approaches the notion of "properties". Rather than burdening the programmer with writing all those verbose getters and setters, the language formalizes the getter/setter contract and generates them for you. For reference, the equivalent class in Groovy looks like this:
```groovy
@Canonical
class UserAndDateTime {
  String username;
  LocalDateTime datetime;
}
```

First, we'll address the point about mutability. Let's get rid of those setters and make our value a real, true value, no more mutable than the number 42:
```java
public final class UserAndDateTime {

  private final String username;
  private final LocalDateTime datetime;

  public UserAndDateTime(String username, LocalDateTime datetime) {
    this.username = username;
    this.datetime = datetime;
  }

  public String getUsername() { return username; }

  public LocalDateTime getDateTime() { return datetime; }

  @Override
  public boolean equals( Object o ) {
    if ( this == o ) return true;
    if ( o == null || getClass() != o.getClass() ) return false;
    UserAndDateTime that = (UserAndDateTime) o;
    return Objects.equals(username, that.username) &&
          Objects.equals(datetime, that.datetime);
  }

  @Override
  public int hashCode() {
    return Objects.hash(username, datetime);
  }
}
```
That's a bit better. How does it look to use it? Let's say we have some sort of `payload` to which we can attach modification metadata. We'd code something like this:
```java
UserAndDateTime modificationMetadata = new UserAndDateTime("Someone", now());
payload.setModificationMetadata(modificationMetadata);
```
And to retrieve that metadata:
```java
payload.getModificationMetadata().getUsername();
payload.getModificationMetadata().getDateTime();
```
Again, it works just fine, but as with our `BoundingBoxCalculator`, the code *reads like code*. There's no need for the code to be this dry to read (unless, of course, tools force you to write code this way or you're stuck in some sort of inter-op situation). What if we aimed for something more fluent and natural and rewrote our class like this?
```java
public final class UserAndDateTime {

  private final String username;
  private final LocalDateTime datetime;

  private UserAndDateTime(String username, LocalDateTime datetime) {
    this.username = username;
    this.datetime = datetime;
  }

  public String by() { return username; }

  public LocalDateTime at() { return datetime; }

  @Override
  public boolean equals( Object o ) {
    if ( this == o ) return true;
    if ( o == null || getClass() != o.getClass() ) return false;
    UserAndDateTime that = (UserAndDateTime) o;
    return Objects.equals(username, that.username) &&
          Objects.equals(datetime, that.datetime);
  }

  @Override
  public int hashCode() {
    return Objects.hash(username, datetime);
  }

  public static Builder by(String username) {
    return new Builder(username);
  }

  public class Builder {

    private String username;

    private Builder(String username) {
      this.username = username;
    }

    public UserAndDateTime at(LocalDateTime datetime) {
      return new UserAndDateTime(username, datetime);
    }
  }
}
```
We've written more code here to implement the `UserAndDateTime` class, but bear with me. Let's assume also that `payload` got rid of it's very verbose `getModificationMetadata()` and `setModificationMetadata(UserAndDateTime)` methods and instead replaced them with the sleeker `modified()` and `modified(UserAndDateTime)` pair. Now the code looks like this (with a little help from our good friend, static import):
```java
import static UserAndDateTime.by;
...

payload.modified(by("Someone").at(now()));

payload.modified().by();
payload.modified().at();
```
Much better!

## Final Remarks
Of course, what appeals to me may not appeal to everyone. The goal is to find a way to write your objects and values such that they are as expressive as possible and fit the domain you are working in. Ideally, the further you get from the really detailed implementation bits, the more declarative your code should be: focus on *what* the code is doing until you really have to write the *how*. It can be surprising how many levels down the abstraction rabbit hole you can go before you have to write very "computery" code.

Don't be afraid to experiment with different ways to obtain the most fluent and readable set of calls. Sometimes, static imports work really well, as was the case with the `by(...).at(...)` construct. Sometimes, you'll want to avoid static imports and instead use the class name as part of the code, as in `BoundingBoxCalculator.withPaddingOf(...)`. Which to choose is really more a matter of style and preference, and you shouldn't be afraid to refactor to improve readability if your initial choice doesn't hold up. There's no one right way to solve a programming problem, but there is a wrong way: the way that favours the computer over the programmer. In other words, don't just write code in your programming language. Use the basic building blocks of your language to create new abstractions, and then use those abstractions to build castles in the sky.
