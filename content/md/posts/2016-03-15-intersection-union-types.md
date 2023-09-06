{
:title "Intersection and Union Types"
:tags ["Design" "Java"]
:toc true
:description "Buckle up for a trip down the rabbit hole! What are <em>intersection types</em> and <em>union types</em>, can we use them in Java, and why would we want to?"
}

For this article, we're going to work through a bit of a thought experiment, at least as far as the Java language is concerned. The aim is to explore what is available to us through the type system, specifically with regards to *intersection types* and *union types*, and to see how building software using these types can change the way we look at certain abstractions or language constructs. Down the rabbit hole we go!

## Intersection Types
### What is an Intersection Type?
An **intersection type** is a type obtained by combining two or more types. A type, `T`, is said to be *assignable* to an intersection type if `T` is assignable to *all* the types in that intersection. To make this clear, consider the following types:
```java
interface Eat {
  void eat();
}

interface Fly {
  void fly();
}

class Pigeon implements Eat, Fly {

  @Override
  public void eat() {
    System.out.println("Eating hotdog crumbs off the street.");
  }

  @Override
  public void fly() {
    System.out.println("Flying at unsuspecting pedestrians.");
  }
}
```
Here we can say that the type `Pigeon` is assignable to the intersection type `Eat & Fly` because it implements both of those interfaces. `Pigeon` is also of course assignable to the individual types `Eat` and `Fly`. Assuming all the following code was valid Java, that means all these statements are true:
```java
Eat thingThatEats = new Pigeon();
Fly thingThatFlies = new Pigeon();
Eat&Fly thingThatEatsAndFlies = new Pigeon();
```
This makes sense conceptually, but can we actually represent this concept in Java? Our options are limited by the type system, but there is one place where we *can* define intersection types: generic type parameters.
```java
<T extends Eat & Fly> void doSomethingWithThingThatEatsAndFlies(T thing) {
  thing.eat();
  thing.fly();
}
```
Now that we know *how* we can define such a type, let's explore *why* we would want to.

### Interface Segregation
**Interface segregation** is the **I** of the SOLID principles. Put simply, the principle encourages us to prefer many small interfaces over large ones. The goal is to promote high cohesion and low coupling within the system: high cohesion because small interfaces should be very specialized, and low coupling because client code can depend only on the specific (and narrow) interfaces it requires to function.

> SOLID is a mnemonic acronym for some of the most important principles of object-oriented software design. The principles are: Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion.

Keeping the interface segregation principle in mind, we want to create types that are as specialized as possible in order to avoid bloated dependencies. Intersection types can help us with this by allowing us to specify *exactly* what we require, and nothing more. Consider, for example, how we might implement the example above without intersection types:
```java
interface Eat {
  void eat();
}

interface Fly {
  void fly();
}

interface Bird extends Eat, Fly {}

class Pigeon implements Bird {

  @Override
  public void eat() {
    System.out.println("Eating hotdog crumbs off the street.");
  }

  @Override
  public void fly() {
    System.out.println("Flying at unsuspecting pedestrians.");
  }
}

void doSomethingWithThingThatEatsAndFlies(Bird bird) {
  bird.eat();
  bird.fly();
}
```
Here we manage to get the same behaviour, but at the expense of introducing a new type to our hierarchy: `Bird`. This is troublesome for two reasons. First, if `Bird` defines additional behaviours beyond the ones it inherits from `Eat` and `Fly`, our method `doSomethingWithThingThatEatsAndFlies()` has the opportunity to make use of those behaviours. We have opened the door wider for ourselves than we should have. This is effectively the same situation as when we code to implementations instead of interfaces, or accept a `List` when a `Collection` or even an `Iterable` would do fine. We want to avoid this so that we are not coupled to a type narrower than what we absolutely need.

Second, introducing the `Bird` type seems more an exercise in working around the limitations of the type system rather than trying to model something in our system correctly. Is it true that every `Bird` can `Eat` and `Fly`? What about ostriches and kiwis? Is it also true that everything that can `Eat` and `Fly` must necessarily be a `Bird`?

By working with intersection types, our intent is as clear as possible: we explicitly care only about such types that can both `Eat` and `Fly`, whatever those types may be.

We now have an idea of how to make our intent for a type as narrow and explicit as possible. Let's turn our attention to *union types* to consider the other direction: how to make explicit all the possible types we can deal with.

## Union Types
### What is a Union Type?
A **union type** is a type that represents an *or* relationship between multiple independent types. Consequently, it is only assignable to types that are the super type of *all* the unioned types. Let's imagine, for the moment, that we could define a union type `Employee | Team`, similar to how above we defined an intersection type of `Eat & Fly`. Consider, for example, a situation where we may want to perform a lookup on an organizational chart and could find either an `Employee` or a `Team`. We can be explicit about what we might get back by using a union type:
```Java
Employee|Team findById(long id) {
  ...
}

Employee|Team result = findById(12345);
if (result instanceof Employee) {
  Employee e = (Employee) result;
  e.fire();
} else if (result instanceof Team) {
  Team t = (Team) result;
  t.disband();
}
```
Of course, we now need `instanceof` checks and casts, but the union type offers us more safety over the alternative. If we had written this without union types, we may be forced to simply return an `Object` from the `findById(long)` method. We would still have to deal with the `instanceof` checks and casts, but we would have no help from the compiler to make sure we were testing for the correct types.

> We could do a bit better if we had more powerful language features available to us, such as implicit casts after an `instanceof` check, or pattern matching on types. Both of these are out of scope for this article, but, for reference, here's the same example as above in Ceylon. Note also that the Ceylon compiler can verify that our `switch` is exhaustive.
```ceylon
Employee|Team result = findById(12345);
switch (result)
case (is Employee) { result.fire(); }
case (is Team) { result.disband(); }
```

This example is a little contrived, so now that we understand what union types are conceptually, let's dig a little deeper a consider a case that is much more practical: *how to be explicit about missing values*.

### Modeling the Absence of a Value
One example of a very useful union type is the union of some type `T` and `None`. Such a union type could be used to model the potential presence or absence of a value. Unfortunately, within the Java type system, there is no `None` or `Null` type to model absence, but there is a type that was introduced in Java 8 (and in earlier versions via third party libraries) which approximates this idea: `Optional<T>`. I say *approximates* because, despite capturing the concept of `T | Null`, an `Optional<T>` does not have direct support in the Java type system; it is just a class like any other. For example, I could still undermine the whole point of `Optional<T>` by returning `null` instead, since `null` is assignable to any type.

> In languages like Ceylon or Kotlin, `String?` and `String` are two effectively distinct types, with the former being nullable and that latter guaranteed to never be `null`. This allows the language to do some enforcement and eliminates the possibility of the annoying null pointer exception.

> There are Java annotations (`@Nonnull` and `@Nullable`) provided by third party libraries that can be used to get some compiler assistance, but as with `Optional<T>`, this can be circumvented.

So, under what circumstances is a type like `Optional<T>` useful? Consider a common situation: searching a collection for a result. Imagine we have an `AddressBook` which contains `Contact`s, and we can look up a `Contact` given some sort of search criteria:
```java
class Contact {
  private final String firstName;
  private final String lastName;
  private final String email;

  Contact(String firstName, String lastName) {
    this(firstName, lastName, null);
  }

  Contact(String firstName, String lastName, String email) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.email = email;
  }

  String firstName() { return firstName; }
  String lastName() { return lastName; }
  String email() { return email; }
}

class AddressBook {
  private final Collection<Contact> contacts = new ArrayList<>();

  AddressBook add(Contact contact) {
    contacts.add(contact);
    return this;
  }

  Contact findByLastName(String lastName) {
    for (Contact c : contacts) {
      if (Objects.equals(c.lastName(), lastName)) {
        return c;
      }
    }

    return null;
  }
}

AddressBook addressBook =
      new AddressBook()
          .add(new Contact(
              "Jason",
              "Bullers",
              "jason.bullers@gmail.com"
          ))
          .add(new Contact(
              "Bob",
              "Loblaw",
              "bob.loblaw@blahblah.ca"
          ));

mailer.sendEmailTo(addressBook.findByLastName("Bullers").email());
```
This looks okay until we realize that extracting the `email` from our contact might throw a `NullPointerException`, because a matching contact may not have been found! Even if we find a contact, that contact's email might be `null` and could cause `sendEmailTo` to fail. We definitely don't want that to happen, so we fix the code:
```java
Contact bullers = addressBook.findByLastName("Bullers");
if (bullers != null) {
  String email = bullers.email();
  if (email != null) {
    mailer.sendEmailTo(email);
  }  
}
```
There are two problems with this solution. First is that it adds a lot of clutter. Second, and more importantly, **nothing about the `findByLastName(String)` or `email()` method signatures told us we should be handling the `null` case**. This is a very dangerous place to find ourselves in, because we are now coding either to javadocs or to implementation details rather than the compiler-enforced contract set out by the API. How can `Optional<T>` help us out of this situation?

Recall our problem: we are trying to find a `Contact` by last name, but may not have a match. This is exactly what the union type `Contact | Null` expresses! Let's update our implementation of `email()` accordingly:
```java
Optional<String> email() { return Optional.ofNullable(email); }
```
Let's also try writing the `findByLastName(String)` method again:
```java
Optional<Contact> findByLastName(String lastName) {
  for (Contact c : contacts) {
    if (Objects.equals(c.lastName(), lastName)) {
      return Optional.of(c);
    }
  }

  return Optional.empty();
}
```
The change to the implementation is minor, but the impact on client code is profound. We no longer get back a `Contact`. Instead, we get back a type that *clearly represents the fact that our query may not have a result*. We are now forced to deal with this situation explicitly before we can do anything with the result:
```java
Optional<Contact> bullers = addressBook.findByLastName("Bullers");
if (bullers.isPresent()) {
  Optional<String> email = bullers.get().email();
  if (email.isPresent()) {
    mailer.sendEmailTo(email);
  }
}
```
This solves our second problem, but it looks like the first problem still remains. This code may be safer, but is just as verbose as before. Can we do better?

### A Context for Computation
We'll come back to the general idea of union types later, but right now let's take a little detour and have a closer look at `Optional<T>` and how we can solve the verbosity problem we saw earlier.

`Optional<T>` is an example of what is know in the world of functional programming as a **monad**. This is a very scary academic sounding word that has very academic and precise definitions attached to it, most of which still elude me. The good news is, in order to make use of one, we only really need to understand that **a monad is a construct that puts a value in a computational context**. It does this by providing two important methods:
1. A method to wrap a value in the context
2. A method to transform that value within the context

For the implementation of `Optional<T>` we have to work with, the first method is provided by `Optional.of(T)` and the second by `Optional.flatMap(Function<? super T, Optional<U>>)`. Several other useful methods are also available, such as `Optional.empty()` to create an `Optional<T>` with no value, `Optional.orElse(T)` to extract the value or a default, and `Optional.ifPresent(Consumer<? super T>)` which passes the wrapped value to the specified `Consumer` if present.

Consider the methods described above. There are three distinct categories of method here: creation (wrapping a value in the context), transformation (mapping the value, if it exists, to a new value), and termination (consuming or extracting the value, if it exists). We can combine these methods and compose behaviour, effectively creating an execution pipeline:
```java
addressBook.findByLastName("Bullers")
           .flatMap(Contact::email)
           .ifPresent(emailer::sendEmailTo);
```
As long as the value remains wrapped in the context, it is the context's responsibility to deal with the presence or absence of the value; we need only care about the computation pipeline we want to put that value through and so can work declaratively instead of imperatively. This helps keep the level of abstraction away from the detailed "how" and closer to the problem domain.

### Exceptional Situations and Union Types
Let's consider another situation where union types might be useful: failure cases. We'll look at how Java models failure and then use the idea of union types and our brief exposure to monads to explore an alternative.

In Java, there are two general categories of exception: **checked exceptions** and **unchecked exceptions**. *Checked* exceptions include exceptions such as `IOException`; they must either be caught or declared as thrown. *Unchecked* exceptions include both runtime exceptions, such as `IllegalArgumentException` or `NullPointerException`, and errors, such as `OutOfMemoryError`; they do not need to be caught or declared. For the rest of this discussion, we will consider only *checked* exceptions and take the hard line that runtime exceptions indicate programmer error and should be allowed bubble up.

So how do checked exceptions work? As mentioned above, we are forced to either catch them or declare that they are being thrown:
```java
long sumIntegersIn(File file) throws FileNotFoundException {
  Scanner scanner = new Scanner(file);
  long sum = 0;
  while (scanner.hasNextInt()) {
    sum += scanner.nextInt();
  }
  return sum;
}

long sum;
try {
  sum = sumIntegersIn(new File("input.txt"));
} catch (FileNotFoundException e) {
  sum = 0;
}

System.out.println(sum * 2);
```
There are two things that can be a little troublesome with the construct the language provides us. First, the logic for catching and handling the exception is noisy. Arguably, it may be a better choice to allow exceptions to bubble up to a higher level where they can be handled meaningfully and in a consolidated fashion to avoid the "cross-cutting concern" of error handling to clutter the code. This is not always feasible or appropriate, however.

Second, the return type of the method `sumIntegersIn` is actually split in two pieces: it can both return a `long` on successful execution and it can "return" a `FileNotFoundException` on failure. These two types are declared in different places.

If we consider union types again, we could think of this method as if it were defined as `FileNotFoundException|Long sumIntegersIn(File file)`. How might we represent such a type?

> As with intersection types, there actually is a place in Java where union types are supported: the multi-catch block. This construct allows multiple exceptions to be handled in the same way:
```java
try {
  // Something dangerous
} catch (IOException|SQLException e) {
  // Handle the exception
}
```

Let's create a new type, `Try<T>`. This type can wrap the results of a success (a value of type `T`) or wrap an exception representing the failure. We'll build it as a monad so that we can build a pipeline to transform the value, if present, or else handle or propagate the failure. Note this implementation of `Try<T>` is nowhere near complete. The goal here is simply to explore what is possible.

First, we'll define a few interfaces. The functional interfaces provided in Java 8 are not sufficient for our purposes because those interfaces do not declare any exceptions. This means that we would be forced to catch any exceptions and wrap them as runtime exceptions if we want to throw them. This would just clutter things.
```java
interface TrySupplier<T>{
  T get() throws Exception;
}

interface TryFunction<T, R> {
  R apply(T t) throws Exception;
}
```
Next, we'll create a very minimal implementation of `Try<T>`:
```java
abstract class Try<T> {
  static <T> Try<T> ofFailable(TrySupplier<T> supplier) {
    try {
      return new Success<>(supplier.get());
    } catch (Exception t) {
      return new Failure<>(t);
    }
  }

  abstract <R> Try<R> map(TryFunction<T, R> function);

  abstract T orElse(T other);

  private static class Success<T> extends Try<T> {
    private final T value;

    private Success(T value) {
      this.value = value;
    }

    @Override
    <R> Try<R> map(TryFunction<T, R> function) {
      try {
        return new Success<>(function.apply(value));
      } catch (Exception e) {
        return new Failure<>(e);
      }
    }

    @Override
    T orElse(T other) {
      return value;
    }
  }

  private static class Failure<T> extends Try<T> {
    private final Exception exception;

    private Failure(Exception exception) {
      this.exception = exception;
    }

    @Override
    <R> Try<R> map(TryFunction<T, R> function) {
      return this;
    }

    @Override
    T orElse(T other) {
      return other;
    }
  }
}
```
And finally, we'll use this new type in place of checked exceptions:
```java
Try<Long> sumIntegersIn(File file) {
  return Try.ofFailable(() -> new Scanner(file))
            .map(this::sumIntegersFrom);
}

private long sumIntegersFrom(Scanner scanner) {
  long sum = 0;
  while (scanner.hasNextInt()) {
    sum += scanner.nextInt();
  }
  return sum;
}

long sum = sumIntegersIn(new File("input.txt")).orElse(0L);
System.out.println(sum * 2);
```
This code accomplishes the same thing as the version above: it attempts to read and sum all the integers present in a specified file, and if reading the file fails, sets the sum to zero. This implementation is just as explicit about the potential for failure as the previous version, but the calling code is cleaner; where previously there was a try-catch construct, this version has a call to `orElse(T)`.

If we had no meaningful way to handle the failure case, we could instead return the `Try<T>` in the event of a failure (perhaps by testing the `Try<T>` with an `isFailure()` similar to the `isPresent()` of `Optional<T>`). We could also add functionality to propagate the exception. That might look something like this:
```java
abstract class Try<T> {
  ...

  abstract T orElsePropagateFailure();

  private static class Success<T> extend Try<T> {
    ...

    @Override
    T orElsePropagateFailure() {
      return value;
    }
  }

  private static class Failure<T> extend Try<T> {
    ...

    @Override
    T orElsePropagateFailure() {
      throw new RuntimeException(exception);
    }
  }
}

long sum = sumIntegersIn(new File("input.txt")).orElsePropagateFailure();
System.out.println(sum * 2);
```

## Final Remarks
Intersection and union types open the door for some interesting solutions to common problems faced in object-oriented software development. As we saw above, the Java language does not really offer much in the way of direct support for modeling these kinds of types, but the type system is rich enough to approximate these ideas.

We saw in the examples above that libraries can be used to create union types and even give them monadic properties. Whether or not this approach is always *practical* is debatable, but understanding what a programming language makes possible opens a lot of doors to design alternatives than can expand the expressiveness of the language for your domain. Sometimes, it really does pay to build a better mousetrap.
