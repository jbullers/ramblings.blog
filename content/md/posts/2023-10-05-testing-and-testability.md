{:title "Testing and Testability"
 :toc true
 :tags ["Java" "Testing"]
 :description "I've learned a lot of lessons about testing and testability in Java over the years,
               specifically: what kinds of tests are worth writing, and how (and when) do you design
               for testability without twisting your design into knots for the sake of testing.
               Let's look at a couple of examples."}

I've written about testing [before](/posts/2016-09-09-impossible-test)
in response to some struggles I was having introducing automated tests to my code base at work.
That post focussed mainly on safely making seams in existing code to get it under test,
but didn't really consider the question of what to test or when
(it's also a fairly domain specific example and was geared toward an audience of coworkers).
Over time, I've gained some insight into what kinds of tests were worth writing,
how to structure them, and how to design for testability.

## What Kinds of Tests Are Valuable?

We'll explore this question using my favourite simple example for this topic: a stack.
Consider the following Java class
(the astute reader will notice a bug in the implementation; we'll get to that):

```java
import java.util.ArrayList;
import java.util.List;
import javax.annotation.CheckReturnValue;
import javax.annotation.Nonnull;
import javax.annotation.Nullable;

import static java.util.Objects.requireNonNull;

/**
 * An immutable FIFO data structure.
 * <p>
 * Create a stack with {@code Stack.&lt;Type&gt;empty()} and add elements
 * using {@link #push(E)}. The top of the stack can be retrieved by calling
 * {@link #peek()}, and the top-most element can be removed by calling
 * {@link #pop()}. Note that since this stack implementation is immutable,
 * both {@code peek()} and {@code pop()} must be called to remove and
 * retain a reference to the top-most stack element.
 *
 * @param <E> the type of elements in this stack
 */
public class Stack<E> {

    private static final Stack<?> EMPTY = new Stack<>();

    /**
     * @param <E> the type of elements in this stack
     *
     * @return an empty stack to hold elements of the desired type
     */
    @SuppressWarnings("unchecked")
    public static <E> Stack<E> empty() {
        return (Stack<E>) EMPTY;
    }

    private List<E> elements = new ArrayList<>();

    private Stack() {}

    /**
     * @param element the element to push
     *
     * @return a new stack containing the given element
     */
    @CheckReturnValue
    public Stack<E> push(@Nonnull E element) {
        var stack = new Stack<E>();
        stack.elements = new ArrayList<>(this.elements);
        stack.elements.add(requireNonNull(
              element, "Attempted to add null element to stack"));
        return stack;
    }

    /**
     * @return the top element on the stack,
     * {@code null} if the stack is empty
     */
    public @Nullable E peek() {
        return elements.get(elements.size() - 1);
    }

    /**
     * @return a new stack containing all but the top element
     *
     * @throws IllegalStateException if the stack is empty
     */
    public @Nonnull Stack<E> pop() {
        if (this.elements.isEmpty())
            throw new IllegalStateException("Tried to pop and empty stack");
    
        var stack = new Stack<E>();
        stack.elements = new ArrayList<>(this.elements);
        stack.elements.remove(elements.size() - 1);
        return stack;
    }

    @Override
    public String toString() {
        return "Stack{" +
              "elements=" + elements +
              '}';
    }

    public static void main(String[] args) {
        var stack = Stack.<String>empty()
                         .push("Hello")
                         .push("World");

        System.out.println(stack.peek()); // => World
        System.out.println(stack.pop());  // => Stack{elements=[Hello]}
    }
}
```

What should the tests for this class look like?

### Testing the API

Earlier in my career, I had absorbed from various sources the notion that
"unit testing" was about testing the smallest units, namely methods, in isolation.
Essentially, this meant that tests should have a one-to-one
(or one-to-many for dealing with corner cases)
relationship with the public API of the subject under test.
Let's start there and see what the skeletal structure of the tests looks like:

```java
import org.junit.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.is;

public class StackTests {

    @Test
    public void testPush() {
        // ...
    }
    
    @Test
    public void testPeek() {
        // ...
    }
    
    @Test
    public void testPop() {
        // ...
    }
}
```

Seems sane so far. Let's start filling in the tests, starting with `testPush`.
We'll follow the classic Arrange-Act-Assert
(or Given-When-Then for BDD folks)
pattern so that our tests are easy to follow and work as nice, self-contained examples of usage:

```java
@Test
public void testPush() {
    // Arrange
    var stack = Stack<String>.empty();
    
    // Act
    var updatedStack = stack.push("Hello").push("World");
    
    // Assert
    // Hmmm...
}
```

How do we assert the correct state of the stack here?
We need some way to inspect the stack and see that it contains the elements that we just pushed.
Rigid adherence to the idea of isolating tests to a single public method is already getting us in trouble:
unless we do some bizarre reflection voodoo, we need to `peek` or `toString` our stack
in order to see what it contains.
Let's use `peek`:

```java
@Test
public void testPush() {
    var stack = Stack<String>.empty();
    
    var updatedStack = stack.push("Hello").push("World");
    
    assertThat(updatedStack.peek(), is("World"));
}
```

Alright, that was a little unsettling, but let's move on and see if we can better isolate our other tests.
Next up, `peek`:

```java
@Test
public void testPeek() {
    // Arrange
    var stack = Stack<String>.empty();
    // Hmmm...
    
    // Act
    var topElement = stack.peek();
    
    // Assert
    assertThat(topElement, is("Hello"));
}
```

We've got a similar problem as before:
we want to test `peek` in isolation, but how do we get elements on to the stack in the first place?
We have a few options:

* Reflection voodoo to change the internals of our stack
* An alternative constructor that takes an existing collection and adds all its elements to the stack
* Call `push` to add elements to the stack

Let's first consider the alternative constructor.
This would definitely work, but it brings along a whole design activity
and changes the surface area of our API;
we have to answer the question: in what order do the elements of the supplied collection appear on the stack?
Worse, we're now considering an API change specifically because of testing.
Yes, test code should be treated as client code, but in this case,
it's not strictly necessary to have this alternative constructor.
DHH has referred to this idea as [test-induced design damage](https://martinfowler.com/articles/is-tdd-dead/):
the notion that dogmatically following certain testing practices can lead away from the API you *want*
and toward an API that's "easy to test".

Let's avoid changing our API, and instead use `push` to get our stack in the right state:

```java
@Test
public void testPeek() {
    var stack = Stack<String>.empty().push("Hello");
    
    var topElement = stack.peek();
    
    assertThat(topElement, is("Hello"));
}
```

Notice anything interesting? This is essentially the exact same test we already wrote for `testPush`!
Notice also that if we were writing the tests before the implementation, we'd be in the same boat.
So is the problem testing or TDD?
No, the problem, like many problems in software development, is **framing**.

### Testing Behaviour

Let's take a step back and consider what it is we're trying to accomplish by testing.
"Testing" is probably one of the worst named concepts in our industry[^1]
because it covers so much more than just proving correctness.
Tests can:
* Help you explore and learn about the problem space or solution space
* Design your system's components and APIs
* Verify and validate assumptions, both technical and business
* Protect against regression from future changes
* Act as documentation for future developers

Through this lens, we see that tests fall into two broad categories:
* Tests that aid in development, but you can throw away later; and
* Tests that you keep and run regularly

Here, we're particular interested in the ones that we keep.
If these tests are sticking around long-term, then we want to make sure they
**clearly communicate what the subject under test should do**.
Moreover, they should be focussed on intended *behaviour* rather than mechanism.
This makes the tests less brittle and aligns them well with the goals of
design aid, validator, and documentation.

What does that look like?

As a simple mnemonic, I like to start all my test names with `should`.
For example, we would say "a stack should..." and write the tests that fill in the blanks.
This reminds me to frame my tests around the desired behaviour of the subject under test
rather than mechanically writing a test for each public method.
An interesting side-effect of framing things this way is that
it moves away from the very rigid "unit means method" notion we started with earlier.
Kent Beck himself has been intentionally vague on what exactly a "unit" is,
and I think this is the reason: a testable unit is very much dependent on context.
Here, the unit is arguably the stack, not its individual methods.

```java
import org.junit.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.is;

public class StackTests {

    @Test
    public void shouldPeekTheLastItemAdded() {
        var stack = Stack.<String>empty()
                         .push("Hello")
                         .push("World");

        assertThat(stack.peek(), is("World"));
    }

    @Test
    public void shouldReturnANewStackLessTheTopItemWhenPopped() {
        var stack = Stack.<String>empty()
                         .push("Hello")
                         .push("World");

        var poppedStack = stack.pop();

        assertThat(poppedStack.peek(), is("Hello"));
    }
}
```

Together, these two tests cover `peek`, `push`, and `pop`.
More importantly, they do it in a way that clearly documents the intended behaviour of the stack,
and surfaces the interconnections between the API methods
(arguably, we only really need the second scenario since it covers all three methods,
but I find the added clarity of documented behaviour worth the functional overlap).

### Squashing Bugs

So far, so good. But what happens if we `peek` an empty stack?

```java
Stack.<String>empty().peek();
// Exception in thread "main" java.lang.IndexOutOfBoundsException:
//   Index -1 out of bounds for length 0
```

Oops. While the signature of `peek` promises a `null` return when the stack is empty,
the implementation forgets to deal with that case and blows up.

Let's imagine that our stack was released out into the wild
and we learned about this problem via a bug report.
A good approach to fixing bugs is to try and reproduce the bug with a (failing) test,
and then fix the problem and see that all tests pass.
What should the test look like?

The following may sound silly in this particular example,
but I've encountered situations in the past where developers have captured
the reported problem one-to-one with a test
and written something like this:

```java
@Test
public void shouldNotThrowIndexOutOfBoundsWhenPeekingAnEmptyStack() {
    var stack = Stack.empty();
    
    // If this errors, the test fails
    stack.peek();
}
```

Technically, this is true: our stack certainly *should not* throw an exception in this situation.
This isn't particularly useful though, because there's a lot of things our stack *should not* do when we `peek`.
That's not to say that negative tests are useless:
there are situations where you genuinely do need to test something does not happen in order to prove correctness.
For example, a listener *should not* be notified if an update did not change the value.
This isn't one of those cases, however, and so we should focus instead on what the stack *should* do:

```java
@Test
public void shouldReturnNullWhenPeekingAnEmptyStack() {
    var stack = Stack.<String>empty();

    assertThat(stack.peek(), is(nullValue()));
}
```

With this test in place, we can easily fix out problem:

```java
/**
 * @return the top element on the stack,
 * {@code null} if the stack is empty
 */
public @Nullable E peek() {
    return elements.isEmpty() ?
           null :
           elements.get(elements.size() - 1);
}
```

For completeness, we should also cover the case of calling `pop` on an empty stack.
Again, we frame this in terms of expected behaviour:

```java
@Test(expected = IllegalStateException.class)
public void shouldThrowIllegalStateExceptionWhenPoppingAnEmptyStack() {
    Stack.empty().pop();
}
```

## Designing for Testability

[^1]: Don't get me started on "sprints".
The team is supposed to work at a pace that they can sustain indefinitely.
We have a word for long races at a steady pace, and they're not "sprints."
Language has a huge impact on framing.
