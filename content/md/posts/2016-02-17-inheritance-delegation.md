{
:title "Inheritance and Delegation"
:tags ["Design" "Java"]
:toc true
:description "We learn a lot about inheritance, and our languages often have constructs that lead us to using it as the solution to many modeling problems. But then we're also told we should prefer delegation to inheritance. What gives?"
}

I was originally going to title this article "Inheritance Versus Delegation," but as I thought more on what I wanted to write about, I realized that using that kind of wording implied that the two were mutually exclusive or at odds in some way. In Java at least, it may even seem this way for two reasons:
1. Java does not have first-class support for delegation.
2. Java has a special language construct to support inheritance through extension (the `extends` keyword).

We'll come back to that first point later. First, we'll take a closer look at inheritance. Specifically, we'll look at how Java handles inheritance and tease that apart to get a clearer picture of what inheritance really is.

## What is Inheritance?
In my experience, many Java programmers conflate the concept of inheritance with the `extends` keyword. I suspect this may be a combination of factors: instructional information that fails to divorce Object Oriented concepts from the tool (language) used to teach them (I certainly encountered that myself in University courses and texts), and the fact that the Java language practically force feeds the programmer a very narrow and opinionated view of inheritance via its choice of keywords and language support.

So then, what *is* inheritance if it's not just one class extending another? Of course, we all know the book definition (*inheritance models an "is a" relationship*) and we've all seen some variant of the toy interview example with animals or shapes. Here's one that deals with polygons:
```java
abstract class Polygonal {

  protected List<Point> coordinates;

  protected Polygonal(List<Point> coordinates) {
    this.coordinates = new ArrayList<>(coordinates);
  }

  double perimeter() {
    // Sum the lengths between pairs of points
    return perimeter;
  }

  abstract Point centroid();
}

class Square extends Polygonal {

  Square(List<Point> coordinates) {
    super(coordinates);
  }

  @Override
  Point centroid() {
    // Find midpoint between coordinates[0] and coordinates[2]
    // (the diagonal). The midpoint is the centroid.
    return centroid;
  }
}
```

Clearly, we can say that a `Square` *is* `Polygonal`. But what's *really* going on here? In this simple example, we are actually seeing three "flavours" of inheritance at work, all rolled up together: *inheritance of state*, *inheritance of type*, and *inheritance of implementation*.

### Inheritance of State
**Inheritance of state** allows a subclass to take on all the state (the fields) of its parent class. In our example, `Square` inherits the `coordinates` field from its parent class `Polygonal`. In order for state to be inherited, that state must be visible to the subclass. In other words, any state that is not private is inherited.

> In Java, this kind of inheritance is only really available to us through the `extends` keyword. Consequently, it is limited to single inheritance. This is at least partly because collisions of fields can get very messy.

### Inheritance of Type
**Inheritance of type** allows a class to take on more than one type. A class can take on as many types as it likes, since the types only define how objects of that class can be treated: what their capabilities are. In our example, `Square` has two types: it is both a `Square` and `Polygonal`. Here, it has three types:
```java
class Square extends Polygonal implements Colored {
  ...
}
```

There's really no way to divorce *inheritance of type* from the concept of inheritance. This is the meat of it, where the *is a* relationship is defined.

> In Java, this kind of inheritance is available to us through both the `extends` and `implements` keywords, where both *classes* and *interfaces* define the types.

### Inheritance of Implementation
**Inheritance of implementation** allows a derived type to take on the *behaviour* of its super types. This is useful because it provides a mechanism for **code sharing** between a parent class and its subclasses. In our example, `Square` inherits the implementation of the `perimeter()` method from the abstract base class `Polygonal`.

> In Java, this used to be limited to single inheritance since the only mechanism for inheritance of implementation was through the `extends` keyword. As of Java 8 however, multiple inheritance of implementation is allowed via the `implements` keyword and default methods (this is a similar idea to mixins in other languages).

It's important to note that, despite the fact that the `extends` keyword applies all three flavours of inheritance in one easy-to-use construct, **all three flavours can be seen as distinct**. The distinction between these flavours of inheritance is crucial to understanding the power and flexibility we get from using delegation in our designs. We'll talk about that next.

> In Java, the evolution of the `implements` keyword makes this a little more obvious: it used to only allow for inheritance of type, it now also allows for inheritance of behaviour, but it **does not allow for inheritance of state**.

## What is Delegation?
Simply put, delegation is when an object asks another object to respond to a message on its behalf. As a simple example, consider a class that represents some piece of data that can be saved, say a `User`. This class may delegate the actual saving of data to a `Persistor`:
```java
class User {

  private Persistor persistor;
  private String userName;
  private String email;

  User(Persistor persistor, String userName, String email) {
    this.persistor = persistor;
    this.userName = userName;
    this.email = email;
  }

  void save() {
    persistor.save(userName, email);
  }
}
```

Okay, great. Simple enough, but what does this have to do with inheritance? Let's revisit the example of `Square` and `Polygonal`, but this time, keeping in mind the three flavours of inheritance. As an end result, we want to have a `Square` that behaves the exact same way from the client programmer's perspective, but is implemented by specifically applying the flavours of inheritance that are necessary. Can we do it?
```java
interface Polygonal {
  double perimeter();
  Point centroid();
}

class ConvexPolygon implements Polygonal {

  private List<Point> coordinates;

  ConvexPolygon(List<Point> coordinates) {
    this.coordinates = new ArrayList<>(coordinates);
  }

  List<Point> coordinates() {
    return unmodifiableList(coordinates);
  }

  @Override
  public double perimeter() {
    // Sum the lengths between pairs of points
    return perimeter;
  }

  @Override
  public Point centroid() {
    // General algorithm for finding the centroid of a convex
    // polygon: sum all the points and divide by the number of
    // points to find the centroid
    return centroid;
  }
}

class Square implements Polygonal {

  private Polygonal delegate;

  Square(List<Point> coordinates) {
    delegate = new ConvexPolygon(coordinates);
  }

  @Override
  public double perimeter() {
    return delegate.perimeter();
  }

  @Override
  Point centroid() {
    // Optimized calculation for squares: find midpoint of
    // delegate.coordinates().get(0) and delegate.coordinates().get(2)
    // (the diagonal). That midpoint is the centroid.
    return centroid;
  }
}
```

In this implementation, where and how each of the flavours of inheritance come in to play is a lot more obvious. *Inheritance of state* has been eliminated (and I would argue should generally be avoided anyway because of its implications on encapsulation, but that's a discussion for another time), *inheritance of type* is made possible via the `implements` keyword, and *inheritance of behaviour* is handled by delegating to the contained `ConvexPolygon` object. From the outside, a `Square` is still `Polygonal`, and we can still ask it for its perimeter and its centroid and get the same result. Success!

## Favour Composition Over Inheritance
So we've rewritten our `Square` in a way that seems more complicated than the original `extends` version. What's the point? The point is that once we understand to look at inheritance through this lens, several doors open for us.

We now have another tool available to us which we can use to solve the problem of code reuse: **composition**. The main benefit of this tool over the traditional inheritance approach is that it is less coupled. I say *less coupled* because inheritance implicitly introduces coupling in the system between a base class and all its derived classes, as well as between the clients of the derived classes and the base class; when the base class changes, the impact is not local (see [Encapsulation and Information Hiding](encapsulation) for more on this subject).

Inheritance also necessarily creates an *is a* relationship that you may not want, but end up with as a side effect of code reuse. Consider a classic example from many Swing GUI programming tutorials:
```java
class Application extends JFrame {

  Application() {
    setContentPane(createContent());
    setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);
    pack();
  }

  private JPanel createContent() {
    JPanel content = new JPanel();
    content.add(new JTextField("Some text"));
    content.add(new JButton("Click me!"));

    return content;
  }

  public static void main(String[] args) {
    SwingUtilities.invokeLater(Application::createAndShowGui);
  }

  private static void createAndShowGui() {
    Application app = new Application();
    app.setVisible(true);
  }
}
```

This works just fine, but is an `Application` *really* a `JFrame`? In most scenarios, the answer is **no**. The `Application` class's single responsibility should be to launch our application; it has no business being a `JFrame` and inheriting all the state and behaviour that comes with that. A better approach would be composition: an `Application` *has a* `JFrame`.
```java
class Application {

  public static void main(String[] args) {
    SwingUtilities.invokeLater(Application::createAndShowGui);
  }

  private static void createAndShowGui() {
    JFrame frame = new JFrame();
    frame.setContentPane(createContent());
    frame.pack();
    frame.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);
    frame.setVisible(true);
  }

  private static JPanel createContent() {
    JPanel content = new JPanel();
    content.add(new JTextField("Some text"));
    content.add(new JButton("Click me!"));

    return content;
  }
}
```

Composition also provides a greater degree of flexibility when it comes to changing the behaviour of an object; a class that was designed to solve the code reuse problem via delegation instead of inheritance can even vary its delegate at run time! This makes for a system that is much easier to configure. It can also lead to a design that is a lot more modular and cohesive, and less coupled.

Consider again the example of a `User` that can be saved. If the saving logic came from a base class, we would very much be tied to that specific implementation:
```java
abstract class Persistable {
  void save() {
    // Write this object out to disk
  }
}

class PersistableUser extends Persistable {

  private String name;
  private String email;

  PersistableUser(String name, String email) {
    this.name = name;
    this.email = email;
  }
}

class Main {
  public static void main(String[] args) {
    new PersistableUser("Jason Bullers", "jason.bullers@gmail.com").save();
  }
}
```

What would we do if we wanted to write user data to a database instead of to disk? In this specific case, the answer is simple: override `save()` and provide a new implementation. But what if we had ten classes that were all `Persistable`, and some needed to be written to disk and some to a database and some across the network? This solution clearly will not scale.

Again, we need to consider what we are trying to model and what flavours of inheritance we really need here. It would be nice to have a type defined so that we could simply call `save()` on all instances of that type. It would also be nice to have code reuse for actually persisting data. Let's try again with composition and delegation:
```java
interface Persistable {
  void save();
}

interface Persistor {
  void save(Object... items);
}

class DiskPersistor implements Persistor {
  @Override
  void save(Object... items) {
    // Write items out to the disk
  }
}

class DatabasePersistor implements Persistor {
  @Override
  void save(Object... items) {
    // Write items out to the database
  }
}

class NetworkPersistor implements Persistor {
  @Override
  void save(Object... items) {
    // Write items out to the network
  }
}

class PersistableUser implements Persistable {

  private Persistor persistor;
  private String name;
  private String email;

  PersistableUser(Persistor persistor, String name, String email) {
    this.persistor = persistor;
    this.name = name;
    this.email = email;
  }

  @Override
  public void save() {
    persistor.save(name, email);
  }
}

class Main {
  public static void main(String[] args) {
    new PersistableUser(
          new DatabasePersistor(),
          "Jason Bullers",
          "jason.bullers@gmail.com"
    ).save();
  }
}
```

You can imagine that we may have another layer in the application that can determine (perhaps as a result of external configuration or user input) which `Persistor` to choose. Clients of our system could even create their own `Persistor`s that we never thought of, and they would work just fine. This kind of dynamic selection would be impossible with the first design that realized inheritance via extension.

## The Downside
One drawback of composition and delegation worth considering was mentioned right up at the beginning of this article: the lack of first-class support for delegation in the Java language. What does this mean?

Consider the example of `Square` and `Polygonal` again. Imagine what the code would look like if there were more methods in `Polygonal` and `Square` chose to delegate more to its contained `ConvexPolygon` object:
```java
interface Polygonal {
  double perimeter();
  double area();
  Point centroid();
  int numberOfSides();
}

class ConvexPolygon implements Polygonal {
  ...
}

class Square implements Polygonal {

  private Polygonal delegate;

  ...

  @Override
  double perimeter() {
    return delegate.perimeter();
  }

  @Override
  double area() {
    return delegate.area();
  }

  @Override
  Point centroid() {
    return delegate.centroid();
  }

  @Override
  int numberOfSides() {
    return delegate.numberOfSides();
  }
}
```

There's a lot of noise here for something so simple as delegating to obtain code reuse. Not only that, but imagine that we decide to add another method, say `numberOfVertices()`, to the `Polygonal` interface and provide an implementation in `ConvexPolygon`. If we have implemented `Square` via inheritance through extension, there would be no problem. However, with this implementation, `Square` would no longer compile until we provided the appropriate boilerplate implementation of the new method:
```java
@Override
int numberOfVertices() {
  return delegate.numberOfVertices();
}
```

With first-class language support (or some other form of compile time code generation), this problem goes away. For example, Kotlin, a JVM language, solves this problem with language support:
```kotlin
// `Square` takes an instance of `Polygonal` called `delegate` and
// implements the Polygonal interface. It forwards all calls to
// `delegate`.
class Square(delegate: Polygonal) : Polygonal by delegate {

  // Square specific things
}
```

Groovy, another JVM language, solves this problem with an annotation:
```groovy
// Square has all the methods of Polygonal and forwards all calls to
// `delegate` automatically
class Square {
  @Delegate Polygonal delegate;

  // Square specific things
}
```

## Final Remarks
Of course, none of this is to say that you should **never** use the `extends` keyword to model inheritance; it, like all other tools the language provides has its place. Recall that `extends` not only models an *is a* relationship, it also provides inheritance of behaviour and inheritance of state all in one. The takeaway here is that you should make a conscious decision to use `extends` instead of automatically reaching for it any time it seems you need to inherit behaviour or type. As we saw, delegation is a perfectly viable option with some significant benefits.

On the other hand, there are some drawbacks to taking the delegation route, especially with the Java language. To a large extent, this drawback is mitigated by keeping interfaces small and cohesive, but there still may be cases where the noise is not worth the added flexibility. While in many cases composition (and delegation) should be preferred, it is by no means the tool to use for every job.
