{
:title "Why Clojure?"
:tags ["clojure"]
:toc true
:description "I've had an on-again, off-again relationship with Clojure for years. Despite not using it in any way at work, and despite many false starts, I keep picking it back up. What keeps me interested in Clojure? Why is it so great? Uncle Bob wrote an article about that a few years back, and I think it misses the mark."
}
Years ago, I discovered Clojure. I remembered Lisp from my University days (we briefly looked at Scheme in one of my courses), and decided that a Lisp on the JVM (I'm a Java developer professionally) would be worth a shot, having already used Groovy a little and read a bit about Scala. Something about Scheme in University tickled me, though we never really spent enough time with it for me to really understand what was so great about it, or why it would be a language used anywhere but in academia. Clojure was touted as a practical and pragmatic language, so there must be something to Lisp.

My first book on the topic was probably not the greatest starting point: I tried reading *Joy of Clojure*. It's a great book, but certainly not one geared toward a Lisp and functional programming newbie. After a few false starts and a YouTube catalog of wonderful Rich Hickey talks[^1], I tried applying functional style and some Clojure Big Ideas to my code and designs at work (e.g. separation of state, identity, and time proved invaluable for a collaborative editing system I was building at the time). Eventually, I decided to climb back up on the Clojure horse and went through *Clojure for the Brave and True*. The quirky humour and general approach to teaching Clojure made it easily digestible. I used my newfound knowledge on a few toy projects, some Advent of Code, and then put Clojure back on the shelf again.

A year or two ago, I took another shot at it: more toy projects, some Exercism problems, and most recently, more Advent of Code. I started listening to some really great podcasts[^2] and watching experienced developers[^3] code on YouTube. And that's where I'm at now: absorbing like a sponge, fiddling around to understand the idioms and get the "feel". I'm not paid to write Clojure code, so why do I keep coming back to it over and over?

Uncle Bob wrote an [article](https://blog.cleancoder.com/uncle-bob/2019/08/22/WhyClojure.html) on what makes Clojure so great. I think it misses the mark.

## What Does Uncle Bob Think?
Uncle Bob argues that the one and only answer to the question "why Clojure?" is: Economy of Expression. He goes on to say that "the language has almost no syntax or grammar," and dives in to a comparison of Clojure and Java code:
```clojure
;; This is Clojure code
(println (take 25 (map #(* % %) (range))))
```
```java
// This is Java code
public class SquaresOfIntegers {
  public static void main(String[] args) {
    for (int i=0; i<25; i++)
      System.out.println(i*i);
  }
}
```
The Clojure code, Uncle Bob says, covers "80% or so of the syntax of Clojure," and it follows a very simple and consistent pattern: the parens denote the beginning and ending of a list, which Clojure interprets as a function call; names are just names; and we just follow along, reading from the inside out to see what is evaluated and what gets passed on to what.

The Java code, on the other hand, is "considerably more wordy" and "covers perhaps 5% of the syntax of Java". He concludes that the minimal syntax of Clojure means that he can "express problems clearly, and directly, with much less effort and contortion than most other languages."

Essentially, Uncle Bob argues that Clojure is, by design, a language that leans heavily into declarative function composition, while Java favours a more verbose, imperative style. This isn't something particularly unique to these two languages, but rather a difference in each language's ancestry: Lisp was always a language that placed functions first, while Java comes from the C/ALGOL-line of languages and so follows their syntactic choices. He also argues, though somewhat superficially and only by implication of the syntax argument, that Clojure provides much better tools for data manipulation given that the language is built on its own data structures.

## Meh
I don't find this argument particularly convincing for two reasons. First, it's an exceptionally subjective metric that is difficult, if not impossible, to disentangle from familiarity and personal preference. For example, `people.add(person)` is arguably more expressive than `(conj people person)`; "economy" walks a line adjacent to overly terse. Consider another JVM language: Groovy. It also provides Economy of Expression, but it doesn't deviate significantly from the syntactic choices that are familiar to Java developers. Here's the same snippet in Groovy:
```groovy
// This is Groovy code
25.times { println it * it }
```
I can also play the same game of explaining just how *easy* it is to read this:
* `25` is just a number
* `times` is just a method we call on the number to do something *number* times
* `{...}` denotes an anonymous function closure
* `it` is an implicit function argument (the current iteration, in this case)

Yes, you can argue percentages of syntax here, but I've honestly never met a working developer who struggles with the syntax of the language they regularly work in. I *have*, however, met developers who have never worked with a Lisp before and struggle to read it, even after the whirlwind syntax tour.

Second, Uncle Bob's argument stands on a constantly shifting landscape. For example, with [JDK 21](https://openjdk.org/jeps/445), the entire program could look like this:
```java
// This is Java code
void main() {
  IntStream.iterate(0, i -> i + 1)
           .map(i -> i * i)
           .limit(25)
           .forEach(System.out::println);
}
```
To be fair, the Clojure program should probably also have a `main` method as its entry point, as it's generally not a good idea to have executable code like this just hanging around top-level:
```clojure
;; This is Clojure code
(defn -main [& args]
  (println (take 25 (map #(* % %) (range)))))
```
or, to avoid reading inside-out, we can use the thread-last macro to create a pipeline of calls, similar to the Java version:
```clojure
;; This is Clojure code
(defn -main [& args]
  (->> (range)
       (map #(* % %))
       (take 25)
       (println)))
```
There's not a huge difference between these, in my opinion: all the snippets above are expressive, and they all avoid unnecessary state management like loop control variables. Preference likely has more to do with familiarity than some arbitrary metric like percentage of syntax shown. If Economy of Expression was all there was, I wouldn't keep coming back to Clojure time and time again; I'd likely be content with a language like Groovy that's both familiar to me as a Java developer and expressive.

I think it's also fair to point out that while Clojure may be light on syntax, there's *a lot* of standard library functions you need to understand in order to use the language effectively. I'd argue that grokking the syntax of a Java `for` loop and grokking higher order functions like `map` aren't wildly different things; once you understand how they work, you know how to read for common patterns.

Clojure also has some interesting "extra syntax" in some places. For example, it has list comprehensions with the `for` macro (which has nothing to do with the classic `for` loop, totally not confusing), so code like this in Python:
```python
# This is Python code
[n**2 for n in range(10) if n % 2 == 0]
```
can be expressed similarly in Clojure:
```clojure
;; This is Clojure code
(for [n (range 10) :when (even? n)] (* n n))
```
Sure, the Clojure version has very simple *language syntax* (we've seen it all already with Uncle Bob's example), but it layers on a mini-DSL that you also have to learn to get the most out of the macro. I'd argue this isn't really any different from the Python construct which is supported by explicit syntax. At a certain point, it's all just variations of the same theme.

> For completeness, Clojure can also do something similar to the Groovy approach:
> ```clojure
> ;; This is Clojure code
> (dotimes [i 25] (println (* i i)))
> ```

## So Why Clojure?
Alright, so if Economy of Expression isn't what's got me hooked, what is? For me, Clojure is a sort of perfect storm of design decisions and features that really resonate with me. I've been thinking on this for a while, trying to see if I can really pin down a couple of things that are the biggest contributors to the overall experience. What it comes down to are two, somewhat related, things: **immutability** and **interactivity**.

### Immutability Baked In
One core design decision Clojure makes is to make immutability a central concept that underpins its entire suite of data types. Strings? Immutable. Vectors? Immutable. Maps? You guessed it: immutable. Functions that transform a data structure, such as by adding a new element, do so by returning the result of the transformation, rather than mutating the data structure in-place.

Why is this interesting? Because so much of what I've learned about defensive programming in Java, even without taking into account the complexities of multi-threading, becomes irrelevant. Here's a simple example of some Java code that I might write (which used to be way more verbose before records were introduced; shifting landscape):
```java
// This is Java code
record Recipe(String name,
              List<Ingredient> ingredients,
              List<String> steps) {
  Recipe {
    // Make some defensive copies of mutable state
    ingredients = List.copyOf(ingredients);
    steps = List.copyOf(steps);
  }
}
```
Maybe this is paranoia, but without the extra step of making defensive copies, the whole idea of immutable records goes out the window. I could hope that whatever code constructs a `Recipe` doesn't do anything with the `ingredients` and `steps` lists after construction, or that callers of `recipe.ingredients()` don't try adding or removing from that list, but it's much less error-prone to enforce that by making the defensive copy (for reference, `List.copyOf` makes an unmodifiable copy, so we're safe on both sides without having to override the accessor methods). Consider Clojure:
```clojure
;; This is Clojure code
(defn recipe [name ingredients steps]
  {:name name, :ingredients ingredients, :steps steps})
  
;; or if you prefer
(defrecord Recipe [name ingredients steps])
```
No defensive copies required, because there's no way to mutate those lists!

To be fair, Uncle Bob's Economy of Expression does come in to play here. Because Clojure has a built-in set of data structures, and because they are all immutable, it's possible to have a feature-rich standard library that works against the simple rule: given a data structure and a transformation, return the transformed data structure. The standard library doesn't have to worry about maybe changing things in place, or maybe making copies, or maybe shallow copies but not deep copies. There's a sane default, and everything is built around that. Consider:
```java
// This is Java code
Recipe addStep(Recipe recipe, String step) {
  var updatedSteps = new ArrayList<String>(recipe.steps());
  updatedSteps.add(step);
  return new Recipe(recipe.name(), recipe.ingredients(), updatedSteps);
}
```
compared to:
```clojure
;; This is Clojure code
(defn add-step [recipe step]
  (update recipe :steps conj step))
```
In the Java version, we're responsible for keeping the immutability guarantee alive. We create a new list of steps that includes the step to add, and we create a new `Recipe` with the same name and ingredients, but with the updated steps; Java can't help us beyond enforcing shallow immutability. Beyond that, `Recipe` is our own custom value type, and so there's no existing behaviour we can use to transform it.

In the Clojure version, we're simply relying on built-in (immutable) data structures and the core function `update` that knows how to transform them. Specifically, given a `recipe` (hash map) and a `step` to add, we `update` the value of the `recipe` at the `:steps` key by `conj`ing (adding) the `step`. It's pretty clear when you're familiar, but I'd argue the real value here is not having to perform mental gymnastics around mutability.

To drive that point home a little, let's see what the Java version would look like if we had the same immutability guarantees that Clojure gives us:
```java
// This is Java code
record Recipe(String name,
              List<Ingredient> ingredients,
              List<String> steps) {

  Recipe addStep(String step) {
    return new Recipe(name(),
                      ingredients(),
                      steps().add(step));
  }
}
```
Is it more verbose? Sure, a little. Is the difference in verbosity enough to make me want to switch to a language with a completely foreign syntax? Nope, especially since my IDE writes half my code for me with auto-complete anyway.

### Interactive Development
Years ago, I watched an amazing talk by Brett Victor titled [Inventing on Principle](https://youtu.be/EGqwXt90ZqA?si=ZGNrM9EvQM2gt4uE). I was absolutely blown away by the technical demonstrations; the entire talk was great, but those parts really stuck with me. As I mentioned earlier, I'm a Java developer professionally. The system I work on is a desktop application, and my particular components allow meteorologists to draw weather phenomena and manipulate them using various graphical tools. Imagine working on a tool like this and encountering a bug, or working on a feature. What do you do? Automated tests or no, you'll want to see the results of your code changes and just try things out for feel. So you start up the application, get everything in the right state, click around, and oh! The bug is still there, or the feature is not quite right. Tear it all down, make some changes, bring it all up again. Rinse and repeat until you're done.

Now imagine after working like that for a few years, you see Brett Victor showing off trees that appear to be dancing, or a platformer protagonist jumping with various heights, all by changing some variables live. No restarting, no loss of state between changes. The running program just adapted to his changes in real-time, allowing him to experiment and explore. He took the programming assembly line I was familiar with and turned it in to sculpting, where the line between what he was building and the stuff it was made of was practically non-existent. What fun it would be to code like that!

Well, guess what Clojure can do? Because everything is immutable and state changes are carefully controlled and very explicit, you don't have to worry about the world changing out from under you when you don't expect it. You're free to capture the old state as you apply transformations, making it easy to roll back to a previous state if you need to, or to visualize the differences. The language is also designed from the ground up to support redefining your globals and your functions. Anything you `def` you can `def` again to be something else.

Not only *can* you work like this, it's *the* way to work. Clojure programmers hype the REPL all the time, but this isn't anything like what Python or JavaScript calls a REPL. It's not a thing on the side you can use to run some experiments, while your code sits over there in your favourite editor, completely separate. In Clojure, your favourite editor starts the REPL, connects to it, and your entire coding experience is **sending code from your editor to the REPL and getting responses back**. You define a function, test it out with some inputs, see something is amiss. You change the implementation, redefine it, try again. At no point in this flow do you need to restart the application. You can capture data from your experimentation to replay later, or save in formal test cases.

It took me a while to really figure this out, but it's a whole new world when you do. For a long time, the overall design of the language and the immutability default were a big draw, as was Rich Hickey and the data-centric ideas that permeate the community. I couldn't shake the feeling that I was missing something though, because I just wasn't familiar enough. That was enough to keep me coming back until I cracked the code and really tried doing some actual REPL-Driven Development. And it was so. Much. Fun.

## Wrapping Up
If you're interested in Clojure, I strongly encourage you to watch experienced developers using their editors to full effect[^4], and to try it out yourself: there's no shortage of great tooling out there for everyone: Calva for VS Code, Cursive for the IntelliJ folks, CIDER for Emacs, and Conjure for NeoVim are some of the ones I know of. The language has a steep learning curve, especially coming from Java-style OOP languages, but I think the experience made me a better developer overall, and the interactivity just made it a wonderful language for exploration. Clojure also has a very helpful and welcoming community; I find the [Clojurians Slack](http://clojurians.net/) particularly great.

If you aren't new to Clojure, I wonder how you would answer the question: why is Clojure the language for you? Do you agree with my assessment? Does it miss the mark for you? Feel free to message me on the Clojurians Slack. I'm always keen to hear and learn from new perspectives.

[^1]: A few of my favourites: [The Value of Values](https://youtu.be/-6BsiVyC1kM?si=dB5VGy2V5Kh9RGqz), [Simple Made Easy](https://youtu.be/LKtk3HCgTa8?si=6Xa3bZ75iFFTUce6), [The Language of the System](https://youtu.be/ROor6_NGIWU?si=tbdzbg1MZpHayEZL)
[^2]: Shout-outs to *Functional Design in Clojure*, *ClojureStream Podcast*, and *defn*
[^3]: Check out [Lambda Island](https://youtube.com/@LambdaIsland?si=_7iH5T15lRX5zP0R) and [emacsrocks](https://youtube.com/@emacsrocks?si=7Z2Z0HYuEu0m0MZL)
[^4]: I've also been enjoying the content produced by [Andrey Fadeev](https://youtube.com/@andrey.fadeev?si=yeUPWCN4IOJvLD8i)
