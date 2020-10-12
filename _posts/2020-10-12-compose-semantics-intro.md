---
layout: post
title: Introduction to Semantics in Jetpack Compose
---
Android developers are familiar with accessibility APIs like `contentDescription` in the View framework to provide accessibility frameworks like TalkBack additional infomration about our applications. How do we do this in Compose?

The answer to that lies in the semantics APIs, which support both accessibility services and testing.

This is the first of two articles that explore the semantics framework. Today we will be taking a high level look at what semantics are and why we need them.

<!--more-->

**Note:** Compose is still in alpha, and these APIs are subject to change.

## What are semantics?

The [Jetpack Compose testing docs](https://developer.android.com/jetpack/compose/testing) give a great introduction to semantics:

> Semantics, as the name implies, give meaning to a piece of UI. In this context, a "piece of UI" (or element) can mean anything from a single composable to a full screen.

Compose and the View framework have fundamentally different architectures. Views are objects with a defined structure, state, and a clear hierarchy after creation, whereas composable functions do not. Once a composeable has emitted its UI there is no way for tools to identify a particular component and interact with it. We need a way to describe our compose UI in a structured way in order for other tools to interact with it.

Semantics provide that structured description of our UI. Every composeable can define semantics, and compose will generate a semantics tree alongside the UI.

As a basic example, consider an icon button that we want to display on screen:

{% highlight kotlin %}
IconButton(onClick = {}) {
  Icon(asset = Icons.Default.Add)
}
{% endhighlight %}

Accessibility services like TalkBack don't know anything about this button- the image could be anything. A visually impaired user might have no idea what the button does! 

Using semantics, we can add a label to the button:

{% highlight kotlin %}
IconButton(
  onClick = {},
  modifier = Modifier.semantics {
    accessibilityLabel = "Add button"
  }
) {
  Icon(asset = Icons.Default.Add)
}
{% endhighlight %}

The `accessibilityLabel` semantic property functions similarly to the old `contentDescription` attribute for Views- it provides a description of the content for accessibility purposes.

## Semantic properties
As we saw in the example above, the key into the world of semantics is [`Modifier.semantics()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/semantics/package-summary#(androidx.compose.ui.Modifier).semantics(kotlin.Boolean,%20kotlin.Function1)).

Typically you will use this with a trailing lambda to configure a [`SemanticsPropertyReceiver`](https://developer.android.com/reference/kotlin/androidx/compose/ui/semantics/SemanticsPropertyReceiver) with the properties that you need.

`SemanticsPropertyReceiver` has a number of properties already defined that you may want to use. Of particular note:

 * `accessibilityLabel` functions like the old `contentDescription`, and I anticipate it will be the most common semantic property. Use this for elements like icons that need a description.
 * `accessibilityValue` describes your element's state. For example, a radio button might set this to either "on" or "off".
 * `testTag` is a convenient way to tag individual elements so you can later access them in your tests. Since composables have no concept of IDs like we had in the View framework, this is the best way to identify specific pieces of UI in your tests.

`Modifier.semantics()` also accepts a `mergeAllDescendants` property, which allows you to group child elements into a single group in the semantics tree. This is useful for cases such as a radio button with text where two composables are two parts of a single element.

Part 2 of this series will go more in-depth on how the available semantic properties affect accessibility.

### Custom semantic properties 
You can also define your own semantics properties! Custom semantic properties are useful for providing additional information about your tests when a single `testTag` string just isn't enough.

Here's an example of how the [Crane](https://github.com/android/compose-samples/tree/1630f6b35ac9e25fb3cd3a64208d7c9afaaaedc5/Crane) compose sample uses custom semantics to enable testing of a more complicated composable. Crane includes a calendar widget that allows users to choose a range of dates.

![Crane calendar widget](/public/assets/posts/semantics_intro/crane_calendar.png){:height="250px"}

When testing this widget is is useful to assert that specific days have specific states applied, for example that "January 21" is the first day that is selected, that "January 23" is the last day selected, and that "January 22" has a middle selection state.

To accomplish this, Crane sets both an `accessibilityLabel` and a custom `dayStatusProperty` on each `Day()`:
{% highlight kotlin %}
Day(
  modifier = Modifier.semantics {
      accessibilityLabel = "${month.name} ${day.value}"
      dayStatusProperty = day.status
  }
  // ...
)
{% endhighlight %}

`dayStatusProperty` is a custom semantic property that allows us to later access the `DaySelectedStatus` enum for a node in the semantics tree. The definition for this custom property looks like this:

{% highlight kotlin %}
val DayStatusKey = SemanticsPropertyKey<DaySelectedStatus>("DayStatusKey")
var SemanticsPropertyReceiver.dayStatusProperty by DayStatusKey
{% endhighlight %}

Now in our tests we can access this property using a semantic matcher
{% highlight kotlin %}
private fun ComposeTestRule.onDateNode(status: DaySelectedStatus) = onNode(
    SemanticsMatcher.expectValue(DayStatusKey, status)
)

@Test
fun testFirstDate() {
    composeTestRule.onDateNode(FirstDay).assertLabelEquals("January 2")
}
{% endhighlight %}

## What's next & resources

Part 2 of this series will explore how screen readers like TalkBack interact with semantics.

The [Jetpack Compose testing docs](https://developer.android.com/jetpack/compose/testing) provide a good introduction to using semantics for testing.

The best place for documentation on Compose's built-in semantic properties is [`SemanticsPropertyReceiver`](https://developer.android.com/reference/kotlin/androidx/compose/ui/semantics/SemanticsPropertyReceiver), which lists both the properties themselves and some helper functions that we will take a look at in part 2.