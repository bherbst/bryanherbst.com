---
layout: post
title: Unit tests with uncaught exceptions in RxJava chains
---
Uncaught exceptions fail JUnit tests, just as we would expect. An uncaught exception in our code will crash the application, so we'd rightfully want uncaught exceptions to fail our unit tests too!

We might also expect that an uncaught exception in an RxJava chain would fail our unit tests. These exceptions cause crashes when they go to the [default error handler](https://medium.com/@bherbst/the-rxjava2-default-error-handler-e50e0cab6f9f), so they should fail our tests too, right?

Unfortunately that is not the case.
<!--more-->

## Why your tests don't fail
Here's a sample test we can run to show this behavior:

{% highlight kotlin %}
@Test
fun rxJavaUncaughtException() {
    Observable.just(1, 2, 3)
        .map { throw Exception("Test failed") }
        .subscribe()
}
{% endhighlight %}

Under the hood, RxJava's default error handler does this with uncaught exceptions:

{% highlight java %}
static void uncaught(@NonNull Throwable error) {
    Thread currentThread = Thread.currentThread();
    UncaughtExceptionHandler handler = currentThread.getUncaughtExceptionHandler();
    handler.uncaughtException(currentThread, error);
}
{% endhighlight %}

This _looks_ fine, but has subtly different behavior in a JUnit test vs. your application. To understand why, we need to know how JUnit fails our tests when they throw an exception. JUnit's test runner actually wraps your code in a try-catch ([source](https://github.com/junit-team/junit4/blob/main/src/main/java/org/junit/runners/ParentRunner.java#L358-L374)) that looks something like this:

{% highlight kotlin %}
try {
    executeYourTestMethod()
} catch (Throwable e) {
    failTest()
}
{% endhighlight %}

RxJava's error handler does not re-throw any exceptions it catches, bypassing JUnit's try-catch and delivering the exception directly to the `Thread.getUncaughtExceptionHandler()`. Thus JUnit has nothing to catch and isn't aware that the test threw an exception.

## How to prevent this in your project

The easiest way to ensure that uncaught exceptions fail your tests is to create a JUnit rule that installs a custom error handler. Here's one that I've been using based on [AutoDispose's implementation](https://github.com/uber/AutoDispose/blob/2.0.0/test-utils/src/main/java/autodispose2/test/RxErrorsRule.java).

{% highlight kotlin %}
class RxJavaUncaughtErrorRule : TestWatcher() {

  private val errors = LinkedBlockingDeque<Throwable>()

  override fun starting(description: Description) {
    RxJavaPlugins.setErrorHandler { t -> errors.add(t) }
  }

  override fun finished(description: Description) {
    RxJavaPlugins.setErrorHandler(null)
  }

  fun getErrors(): List<Throwable> = errors.toList()

  fun hasErrors(): Boolean = errors.peek() != null

  fun assertNoErrors() {
    if (hasErrors()) {
      throw AssertionError(
        "Expected no errors but RxJavaPlugins received " + getErrors()
      )
    }
  }
}
{% endhighlight %}

You can then apply it to your tests like so:

{% highlight kotlin %}
class RxJavaTests {

    @get:Rule
    val uncaughtRxJavaErrors = RxJavaUncaughtErrorRule()

    @Test
    fun `RxJava uncaught exception`() {
        Observable.just(1, 2, 3)
            .map { throw Exception("Test failed") }
            .subscribe()
        
        // This test will now fail as expected!
        uncaughtRxJavaErrors.assertNoErrors()
    }
}
{% endhighlight %}

## What's next?
You can read more about this issue and find a number of other solutions on RxJava's GitHub [here](https://github.com/ReactiveX/RxJava/issues/5234)

The solution I provided above with a JUnit rule can be a bit of a pain because you need to manually apply it to your tests. As recommended by others in the issue thread, you could use a JVM agent to apply a similar solution across your entire project.