{
:title "To Test... the Untestable Code"
:tags ["Testing" "Java"]
:toc true
:description "Maybe you hear it often, or maybe you've said it yourself: <em>I would write an automated test for it if it wasn't impossible to test.</em> Is it impossible though? Let's get creative."
}

Don Quixote reference aside, there are times when we encounter legacy production code that seems to have an uncanny ability to resist our best efforts to get it under test. The result is often very frustrating and demoralizing, and if we give in to those feelings, we'll be falling back to writing manual test cases, or worse, no test cases. Of course, neither of these scenarios is ideal: manual test cases are rarely executed more than once and can quickly get out of date with the reality that is the code base, and no tests are not helpful at all.

The goal of this article is to walk through an example of a recent problem I had to deal with, and to explore at least one way to address the problem of "untestable" code. Of course, there may be other ways to address even the specific problem we'll examine here. I'll try and highlight alternative implementations I considered while getting the code under test, but these alternatives are by no means exhaustive. I leave it as an exercise for the reader to consider other approaches to solving the problem.

Now let's go take down a <del>windmill</del> giant!

## The Problem
The code I was working with comes from a class called the `EditingLegendRequestDispatcher`. The purpose of this class is to act as a bridge between a **legend** that displays some information to the user and a **legend request handler** that provides static or dynamic content for the legend to display.

> Whether or not the design of the dispatcher is even a good idea in the first place would require a *much* longer article. For example, the only injected dependency is a `Legend` when we clearly need access to other components, such as a `Controller`. We'll assume for the purposes of this discussion that there's nothing we can do about that.

Most of the methods in the `EditingLegendRequestDispatcher` follow a similar pattern:
1. Check the editing components of the `Legend`'s `Controller` for any matching handlers.
2. If no handler is found at level, dispatch a "command" to recursively search for handlers in the hierarchy of `Controller`s.

For this article, we'll consider a stripped down version of the `EditingLegendRequestDispatcher` with only a single (representative) method, `getTextContent`:
```java
public class EditingLegendRequestDispatcher extends BaseLegendRequestHandler {

  public EditingLegendRequestDispatcher(Legend legend) {
    super(legend);
  }

  public String getTextContent(String key, String format, LegendElement element) {
    EditingLegendRequestHandler legendRequestHandler =
          findLegendRequestHandlerAtLevel(Mode.TEXTUAL, key);
    if (legendRequestHandler != null) {
      return legendRequestHandler.getTextContent(key, format, element);
    }

    DispatchEditingLegendTextContentRequestCommand command =
          new DispatchEditingLegendTextContentRequestCommand(this, key, format, element);
    dispatchLegendRequest(command);

    return command.getTextContent();
  }

  private void dispatchLegendRequest(PACCommand command) {
    try {
      getLegend().getView().getController().traverseAllChildAgents(command);
    } catch (NjException e) {
      // Handle error
    }
  }
}
```
Let's consider how we would approach testing this method.

## A First Attempt
We'll start with what should be a simple test: assuming that we don't find a handler at level, the dispatched request should find some text content. Let's set up our test and then consider what dependencies we need to mock out in order to isolate our subject under test.
```java
@Test
public void shouldDispatchTextContentRequestToHandler() {
  EditingLegendRequestDispatcher dispatcher = new EditingLegendRequestDispatcher(mock(Legend.class));

  String textContent = dispatcher.getTextContent(key, null, null);

  assertThat(textContent, is(equalTo(expectedTextContent)));
}
```
If we run this test, the `dispatchLegendRequest` call will explode; we've got a "train wreck" series of method calls to mock if we want any hope of this test working as expected. So let's start mocking that chain:
```java
@Mock Legend legend;

@Before
public void setUpMockLegend() {
  Controller controller = mock(Controller.class);
  doNothing().when(controller).traverseAllChildAgents(any(PACCommand.class));

  View view = mock(View.class);
  when(view.getController()).thenReturn(controller);

  when(legend.getView()).thenReturn(view);
}

@Test
public void shouldDispatchTextContentRequestToHandler() {
  EditingLegendRequestDispatcher dispatcher = new EditingLegendRequestDispatcher(legend);

  String textContent = dispatcher.getTextContent(key, null, null);

  assertThat(textContent, is(equalTo(expectedTextContent)));
}
```
Phew! Okay, now that the chain is all mocked out, we can actually run this test. Or can we? What does `DispatchEditingLegendTextContentRequestCommand` do with its reference to our dispatcher? I won't bother diving in to the code, but suffice it to say the command does some reaching of its own. This means we've got a lot more mocking to do if we ever want to get this test running, but should we really be doing that? Consider the implication of setting up our mocks to work correctly with the calls in `DispatchEditingLegendTextContentRequestCommand`: we'd be setting up our mocks for testing the `EditingLegendRequestDispatcher` using *implementation details of an implementation detail*. The resulting test would not only be very complicated to understand, it would also be *extremely* brittle. Something is clearly wrong here, so let's stop and take a step back.

## What Makes This Test So Difficult?
We've seen in the previous section that our test, which seemed on the surface to be rather straightforward, is in fact nearly impossible to write given the current implementation. This is because the **dependencies** of our `EditingLegendRequestDispatcher` are not explicit. The problem manifests itself in two places:
1. The implementation of `getTextContent` requires access to a `Controller`, which it can only obtain through a "train wreck" chain of calls starting from a `Legend`.
2. The `DispatchEditingLegendTextContentRequestCommand` is used to search for the text content. This command is a **collaborator** of our object (and therefore a dependency), but it is hidden away as an implementation detail. This makes mocking it extremely difficult without special tools.

We could get rid of both of these problems by making the dependencies explicit. The resulting code would then look something like this:
```java
@FunctionalInterface
public interface CommandSupplier {
  DispatchEditingLegendTextContentRequestCommand createCommand(
        EditingLegendRequestDispatcher dispatcher, String key,
        String format, LegendElement element);
}

public class EditingLegendRequestDispatcher extends BaseLegendRequestHandler {

  private final Controller controller;
  private final CommandSupplier supplier;

  public EditingLegendRequestDispatcher(
        Legend legend, Controller controller, CommandSupplier supplier) {
    super(legend);
    this.controller = controller;
    this.supplier = supplier;
  }

  public String getTextContent(String key, String format, LegendElement element) {
    EditingLegendRequestHandler legendRequestHandler =
          findLegendRequestHandlerAtLevel(Mode.TEXTUAL, key);
    if (legendRequestHandler != null) {
      return legendRequestHandler.getTextContent(key, format, element);
    }

    DispatchEditingLegendTextContentRequestCommand command =
          supplier.createCommand(this, key, format, element);
    dispatchLegendRequest(command);

    return command.getTextContent();
  }

  private void dispatchLegendRequest(PACCommand command) {
    try {
      controller.traverseAllChildAgents(command);
    } catch (NjException e) {
      // Handle error
    }
  }
}
```
The test is now as straightforward as we imagined it would be:
```java
@Mock DispatchEditingLegendTextContentRequestCommand mockCommand;

@Before
public void setUpMockCommand() {
  when(mockCommand.getTextContent()).thenReturn(expectedTextContent);
}

@Test
public void shouldDispatchTextContentRequestToHandler() {
  EditingLegendRequestDispatcher dispatcher =
        new EditingLegendRequestDispatcher(
              mock(Controller.class),
              (dispatcher, key, format, element) -> mockCommand
        );

  String textContent = dispatcher.getTextContent(key, null, null);

  assertThat(textContent, is(equalTo(expectedTextContent)));
}
```
Now that we know what would be ideal, how can we approximate this given our current constraints?

## A Solution
Since we're working with a larger system and plugging our dispatcher in to a framework that isn't under our direct control, we can't update the constructor as we did above. While the solution we explored is not applicable, the idea still is: we want to make minimal (safe) changes to make the seams between our object and its dependencies explicit. We already have access to the `Controller` and can mock the call chain from the `Legend` easily enough, so we'll ignore that for now and focus on the command.

Let's pull the creation of the command out of `getTextContent` like we did above, but leave the constructor alone. Since we can't inject the supplier, we'll just initialize the field directly.
```java
@FunctionalInterface
public interface CommandSupplier {
  DispatchEditingLegendTextContentRequestCommand createCommand(
        EditingLegendRequestDispatcher dispatcher, String key,
        String format, LegendElement element);
}

public class EditingLegendRequestDispatcher extends BaseLegendRequestHandler {

  private CommandSupplier supplier = (dispatcher, key, format, element) ->
        new DispatchEditingLegendTextContentRequestCommand(dispatcher, key, format, element);

  public EditingLegendRequestDispatcher(Legend legend) {
    super(legend);
  }

  public String getTextContent(String key, String format, LegendElement element) {
    EditingLegendRequestHandler legendRequestHandler =
          findLegendRequestHandlerAtLevel(Mode.TEXTUAL, key);
    if (legendRequestHandler != null) {
      return legendRequestHandler.getTextContent(key, format, element);
    }

    DispatchEditingLegendTextContentRequestCommand command =
          supplier.createCommand(this, key, format, element);
    dispatchLegendRequest(command);

    return command.getTextContent();
  }

  private void dispatchLegendRequest(PACCommand command) {
    try {
      controller.traverseAllChildAgents(command);
    } catch (NjException e) {
      // Handle error
    }
  }
}
```
This may seem a little silly, but consider what this small change has given us: the `getTextContent` method no longer has any collaborators that it *directly creates*. Now that the seam between our dispatcher and the command is explicit, we can inject our mock.

Unfortunately, this is where things start to get a little ugly. We have more or less two choices at this point: we can use reflection in our test code, or we can provide a package-visible setter for our test code to use. Personally, I prefer the reflection approach since I feel that it makes the smelliness a little more obvious; the setter would have to be named something silly like `setCommandSupplierForTestPurposes` in order to have the same effect. The reflection approach also has the benefit of confining the testing "trick" to the test code, as opposed to adding a "for testing only" method to the production code.

Taking the reflection approach, our test would now look something like this:
```java
@Mock Legend legend;
@Mock DispatchEditingLegendTextContentRequestCommand mockCommand;

@Before
public void setUpMockLegend() {
  Controller controller = mock(Controller.class);
  doNothing().when(controller).traverseAllChildAgents(any(PACCommand.class));

  View view = mock(View.class);
  when(view.getController()).thenReturn(controller);

  when(legend.getView()).thenReturn(view);
}

@Before
public void setUpMockCommand() {
  when(mockCommand.getTextContent()).thenReturn(expectedTextContent);
}

@Test
public void shouldDispatchTextContentRequestToHandler() {
  EditingLegendRequestDispatcher dispatcher = new EditingLegendRequestDispatcher(legend);
  setInternalState(dispatcher, "supplier", (dispatcher, key, format, element) -> mockCommand);

  String textContent = dispatcher.getTextContent(key, null, null);

  assertThat(textContent, is(equalTo(expectedTextContent)));
}
```

## Final Remarks
I hope to have demonstrated through the above example that, while testing legacy code can be very difficult sometimes, it is not impossible. A good first step to getting "untestable" code under test is to imagine what you would change if you had full control over the class. What are the dependencies you wish had been injected? What collaborators are being "newed up" as an implementation detail? Answering these and related questions can help you identify what changes you need to make to find the seams. Once you've found those seams, you need to get creative and find ways to exploit them.

It's also worth noting that while reflection can get a little messy and can be brittle (especially when dealing with internal fields by name!), it's not the worst approach in test code: it makes explicit the fact that the test code is forced to jump through hoops due to poor design, and it avoids polluting production code with "test-only" hooks, or methods/fields that have weaker access privilege than they should. Remember: your tests can and should help you *improve* your designs. You shouldn't be making design concessions just to make your code testable.
