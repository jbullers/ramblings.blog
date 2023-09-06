{
:title "Encapsulation and Information Hiding"
:tags ["Design" "Java"]
:toc true
:description "What does it mean to have \"strong encapsulation\" and to \"hide information\"? Are getters and setters enough? An exploration of designing classes that reveal as little as possible."
}

## What is Encapsulation?
Strictly speaking, **encapsulation** is the mechanism whereby *data* and *behaviour* are bundled together as one unit. This mechanism is provided in many object-oriented languages by the `class` construct. For example, consider the following two code snippets. In the first, we model points using simple arrays for the data structure and have a separate function that operates on those data structures:
```java
int[] pointA = {0, 0};
int[] pointB = {3, 4};

static double distanceBetween(int[] a, int[] b) {
  return hypot(b[0] - a[0], b[1] - a.[1]);
}

double distance = distanceBetween(pointA, pointB);
```
In the second, we model a data structure for `Point` objects that also bundles behaviour with the data:
```java
class Point {
  final int x;
  final int y;

  Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  double distanceTo(Point other) {
    return hypot(other.x - this.x, other.y - this.y);
  }
}

double distance = new Point(0, 0).distanceTo(new Point(3, 4));
```
The former is not encapsulated by the above definition, while the latter is. Notice that in neither case do we take advantage of access modifiers, such as `private`, in order to abstract away details. This is the domain of *information hiding*.

## What is Information Hiding?
**Information hiding** is the practice of building abstractions. Many object-oriented languages provide access modifiers as a way to implement information hiding. The value of this technique is in setting up a contract between a piece of code and the user of that code. This concept is easily summed up in pithy phrases we've all heard before, such as "code to interfaces, not implementations".

For the purposes of this article, I'm not going to continue to make a distinction between *encapsulation* and *information hiding*. There's plenty of debate out there on the fine line between the two, but outside of pure academic rigour, I don't feel that the distinction is particularly meaningful; encapsulation without information hiding doesn't do much to aid in good object-oriented design.

## Are Private Fields and Getters and Setters Enough?
So if we can use access modifiers to hide information, does that mean that the following is a properly encapsulated `Point` class?
```java
public class Point {
  private int x;
  private int y;

  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  public int getX() { return x; }

  public int getY() { return y; }

  public double distanceTo(Point other) {
    return hypot(
          other.getX() - this.getX(),
          other.getY() - this.getY()
    );
  }
}
```
Consider instead this implementation:
```java
public class Point {
  public final int x;
  public final int y;

  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  public double distanceTo(Point other) {
    return hypot(other.x - this.x, other.y - this.y);
  }
}
```
Aside from violating the uniform access principle, there is really no difference between these two implementations as far as encapsulation goes. As a client programmer of the `Point` class, it is perfectly obvious what the implementation of the class is: a `Point` has two integer fields named `x` and `y`.

> The uniform access principle states that all services offered by a module should not give away any details about whether they area implemented via storage or computation. In the case of Java, it is clear that this principle is violated when working directly with fields. Since Java does not have first-class property support, code such as `point.x` or `point.y = 5` can only be doing one thing: retrieving or modifying stored values.

> In a language such as Python or Groovy that *does* have first-class property support, the distinction between fields and calculated values melts away: `point.x` may just as well be a calculated value as it is a stored value. It can even change between one or the other over the life time of the class with no impact on clients of that class.

If `private` fields and `public` getters does not make a class encapsulated, what does? Providing an abstraction to the outside world does. Continuing with our `Point` example, we could define a more abstract version of a `Point` that does not give away its internals like so:
```java
public class Point {
  private double x;
  private double y;

  public static Point newCartesianPoint(double x, double y) {
    return new Point(x, y);
  }

  public static Point newPolarPoint(double r, double theta) {
    return new Point(r * cos(theta), r * sin(theta));
  }

  private Point(double x, double y) {
    this.x = x;
    this.y = y;
  }

  public double getX() { return x; }

  public double getY() { return y; }

  public double getR() { return hypot(x, y); }

  public double getTheta() { return atan2(y, x); }

  public double distanceTo(Point other) {
    return hypot(
          other.getX() - this.getX(),
          other.getY() - this.getY()
    );
  }
}
```
Here, we are free to change the implementation to store the components of the point in polar coordinates if we so chose (or some other form). From the client programmer's perspective, there would be absolutely no difference in terms of how they would use this class; they could still create `Point` objects in one of two ways, and could still extract the Cartesian and polar coordinates as desired.

This example is a bit contrived, especially since here we are dealing with a value rather than an object and therefore have less information to hide (see [Objects Versus Values And API Design](objects-vs-values-api)), but hopefully illustrates the basic idea. Let's now consider another example, where we have a `Library` that contains `Book`s:
```java
public class Library {
  private Map<Author, Collection<Book>> authorToBooks;
  private Map<String, Book> titleToBook;

  public Library add(Book book) {
    addBookToAuthorLookup(book);
    addBookToTitleLookup(book);
    return this;
  }

  private void addBookToAuthorLookup(Book book) {
    Collection<Book> booksByAuthor = authorToBooks.get(book.author());
    if (booksByAuthor == null) {
      authorToBooks.put(book.author(), booksByAuthor = new ArrayList<>());
    }
    booksByAuthor.add(book);
  }

  private void addBookToTitleLookup(Book book) {
    titleToBook.put(book.title(), book);
  }

  public Collection<Book> findAllBooksBy(Author author) {
    return new ArrayList<>(authorToBooks.get(author));
  }

  public Book findBookTitled(String title) {
    return titleToBook.get(title);
  }
}
```
For simplicity, we assume one `Author` to a `Book`, which of course is not always the case in the real world. We also assume that all `Book`s have unique titles; in a real system, we'd probably want to key on ISBNs. The point here is that the implementation involves several maps to improve the lookup performance (imagine we have profiled our system and discovered that search operations vastly outweigh insertions), but all of this is hidden from the client code, which knows only that it can insert `Book`s into the `Library` and search for them by `Author` or title.

## Is Encapsulation Only About Classes?
As I mentioned earlier, information hiding is made available through access modifiers. Consequently, this is not limited only to classes, but extends to other forms of code organization that support access modifiers, such as packages or modules. For example, the following classes living in the same package are still encapsulated:
```java
public interface Point {
  double getX();
  double getY();
  double getR();
  double getTheta();

  default double distanceTo(Point other) {
    return hypot(
          other.getX() - this.getX(),
          other.getY() - this.getY()
    );
  }

  static Point newCartesianPoint(double x, double y) {
    return new CartesianPoint(x, y);
  }

  static Point newPolarPoint(double r, double theta) {
    return new PolarPoint(r, theta);
  }
}

class CartesianPoint implements Point {
  private final double x;
  private final double y;

  CartesianPoint(double x, double y) {
    this.x = x;
    this.y = y;
  }

  @Override
  public double getX() { return x; }

  @Override
  public double getY() { return y; }

  @Override
  public double getR() { return hypot(x, y); }

  @Override
  public double getTheta() { return atan2(y, x); }
}

class PolarPoint implements Point {
  private final double r;
  private final double theta;

  PolarPoint(double r, double theta) {
    this.r = r;
    this.theta = theta;
  }

  @Override
  public double getX() { return r * cos(theta); }

  @Override
  public double getY() { return r * sin(theta); }

  @Override
  public double getR() { return r; }

  @Override
  public double getTheta() { return theta; }
}
```
From the outside, a client programmer has no idea what the implementation of `Point` might be, or even whether or not there may be multiple classes that are used to implement `Point`. As we saw with our earlier implementation, we simply chose Cartesian coordinates for storing our `Point` fields, while here we opted to use two concrete implementations instead.

What is important to recognize here is that good encapsulation is about *insulating the client code from the details*. The scope of those details are unimportant, provided they are hidden behind a well-defined interface that does nothing to give them away. This promotes very low coupling between parts of the system and allows for changes in implementation that will not break calling code.

## Encapsulation and Inheritance
What happens now when we start modeling class hierarchies via inheritance? Recall that inheritance can take on several different flavours: *inheritance of state*, *inheritance of type*, and *inheritance of implementation* ([Inheritance and Delegation](inheritance-delegation)). Of these flavours, it is **inheritance of state**, and, to a lesser extent, **inheritance of implementation**, that can get us in to trouble with regards to good encapsulation.

### How Can Inheritance Break Encapsulation?
When we allow for the inheritance of state, we are necessarily coupling our superclass with all potential subclasses. In this situation, we are said to have *broken encapsulation* because we have revealed too much information about our superclass' implementation. Consider the following inheritance hierarchy:
```java
class Employee {
  protected String firstName;
  protected String lastName;
  protected BigDecimal hourlyRate;

  Employee(String firstName, String lastName, BigDecimal hourlyRate) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.hourlyRate = hourlyRate;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate.multiply(hoursWorked);
  }

  @Override
  public String toString() {
    return format("%s, %s", lastName, firstName);
  }
}

class Manager extends Employee {
  protected Collection<Employee> managedEmployees;

  Manager(String firstName, String lastName, BigDecimal hourlyRate) {
    super(firstName, lastName, hourlyRate);
    managedEmployees = new ArrayList<>();
  }

  Manager addEmployee(Employee employee) {
    managedEmployees.add(employee);
    return this;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate.multiply(hoursWorked).add(managerBonus());
  }

  private BigDecimal managerBonus() {
    return new BigDecimal(managedEmployees.size() * 200);
  }

  @Override
  public String toString() {
    return format(
          "%s, %s: Manages: %s",
          lastName, firstName,
          managedEmployees
    );
  }
}
```
At first glance, this may seem like an acceptable implementation: it minimizes boilerplate and provides subclasses access to important fields. However, consider what may happen if any of those details changes in `Employee`. For example, imagine we decided to rewrite `Employee` as follows:
```java
class Employee {
  protected Name name;
  protected BigDecimal hourlyRate;

  Employee(String firstName, String lastName, BigDecimal hourlyRate) {
    name = new Name(firstName, lastName);
    this.hourlyRate = hourlyRate;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate.multiply(hoursWorked);
  }

  @Override
  public String toString() {
    return name.toString();
  }
}
```
Here, the public API is exactly the same, and yet the `Manager` class will no longer compile. Specifically, the `toString()` method is broken because the `protected` fields `lastName` and `firstName` no longer exist. Changes to the superclass have bled through to the subclass and broken its implementation because `Employee` was not properly encapsulated.

We can do better if we avoid *inheritance of state*, but the problem is still a complex one: how can we design a superclass that exposes the *right information* to its subclasses at the *right level of abstraction*? The answer is not always obvious. In the case of our `Employee` and `Manager` example above, we can define `protected` methods and treat those as if they have the same permanence as a `public` API. This at least insulates us from potential changes in state made to the superclass.
```java
class Employee {
  private String firstName;
  private String lastName;
  private BigDecimal hourlyRate;

  Employee(String firstName, String lastName, BigDecimal hourlyRate) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.hourlyRate = hourlyRate;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate.multiply(hoursWorked);
  }

  protected String lastName() { return lastName; }

  protected String firstName() { return firstName; }

  protected BigDecimal hourlyRate() { return hourlyRate; }

  @Override
  public String toString() {
    return name.toString();
  }
}

class Manager extends Employee {
  protected Collection<Employee> managedEmployees;

  Manager(String firstName, String lastName, BigDecimal hourlyRate) {
    super(firstName, lastName, hourlyRate);
    managedEmployees = new ArrayList<>();
  }

  Manager addEmployee(Employee employee) {
    managedEmployees.add(employee);
    return this;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate().multiply(hoursWorked).add(managerBonus());
  }

  private BigDecimal managerBonus() {
    return new BigDecimal(managedEmployees.size() * 200);
  }

  @Override
  public String toString() {
    return format(
          "%s, %s: Manages: %s",
          lastName(), firstName(),
          managedEmployees
    );
  }
}
```
Now, if we make the same implementation change to `Employee`, the `Manager` subclass still compiles and works as expected:
```java
class Employee {
  private Name name;
  private BigDecimal hourlyRate;

  Employee(String firstName, String lastName, BigDecimal hourlyRate) {
    name = new Name(firstName, lastName);
    this.hourlyRate = hourlyRate;
  }

  BigDecimal calculatePayForHoursWorked(BigDecimal hoursWorked) {
    return hourlyRate.multiply(hoursWorked);
  }

  protected String lastName() { return name.last(); }

  protected String firstName() { return name.first(); }

  protected BigDecimal hourlyRate() { return hourlyRate; }

  @Override
  public String toString() {
    return format("%s, %s", lastName(), firstName());
  }
}
```
> It is worth noting once again that languages such as Python and Groovy help solve this problem with first-class property support: changing from field access to a calculation is transparent to callers. For languages without this kind of support, such as Java, it is up to the author of the class to think ahead and encapsulate properly.

## Final Remarks
Encapsulation and, of course, information hiding are invaluable tools in object-oriented design. They make it possible for us to create modular systems that are loosely coupled, which leads to a whole host of benefits from testability to maintenance. However, as we have seen, it is not always obvious where or how to properly encapsulate details, especially when inheritance hierarchies are involved. Special care must be taken to expose only what is absolutely necessary for client programmers to know. Once an API has been published, whether it be `public` or `protected`, any changes to it can have far reaching consequences. Prefer APIs that steer clear of implementation details and favour useful abstractions instead.
