---
layout: post
title: Autofill with Jetpack Compose
---
Autofill is one of my favorite features new Android from the past few years. I never want to manually type in my address and credit card information again. Autofill makes filling out forms an absolute breeze!

With Jetpack Compose on the horizon I've been seeing a lot of questions around how to support autofill in Jetpack Compose. Yes, autofill is supported in Compose, and here's how you use it!

<!--more-->

_Warning: Google intends to simplify interaction with the Autofill framework by supporting an autofill `Modifier` in the future. Since Compose is currently in beta with a goal of maintaining API stability, it is unlikely to change prior to a stable 1.0 release. Keep an eye out for it though!_

### Autofill in Views
Before diving into Compose, lets review how autofill works for Views. Autofill for most Views is pretty straightforward- you just specify an [`android:autofillHint`](https://developer.android.com/reference/androidx/autofill/HintConstants#attr_android:autofillHint) attribute in your XML file or call [`setAutofillHint(String[])`](https://developer.android.com/reference/android/view/View.html#setAutofillHints(java.lang.String...)) in code. 

These hints are free form strings. You can put anything you want in there! However, it is unlikely that any autofill services will be able to correctly identify custom autofill types you make up. You should generally stick to the constants defined in the [androidx.autofill library](https://developer.android.com/reference/androidx/autofill/HintConstants) to ensure you are using standard hints.

Adding an `autofillHint` works well for most Views from the framework. Unfortunately it is easy to get into a situation in which supporting autofill is a bit more work. A common example is when the data you receive from autofill doesn't match the format that you expect. In these cases you may need to subcless the View and override the [`autofill()`](https://developer.android.com/reference/android/view/View#autofill(android.util.SparseArray%3Candroid.view.autofill.AutofillValue%3E)) method to properly handle the incoming data.

## Autofill in Compose
The first thing we need to add autofill support to a Composable is an [`AutofillNode`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillNode).

An `AutofillNode` needs three things:
 * A list of [`AutofillType`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillType)
 * An `onFill` callback that will be called when the user selects an autofill suggestion
 * A `Rect` with the coordinates of the Composable in the window

Note that `AutofillType` is an enum, not a string like it is in the View framework. I see this as a great improvement to ensure everyone is using a consistent set of autofill hints.

The bounding `Rect` is fairly easy to get- we can use [`Modifier.onGloballyPositioned()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/layout/OnGloballyPositionedModifier#ongloballypositioned) to get the Composable's bounds in the window.

A complete `AutofillNode` will look like this:
```kotlin
AutofillNode(
  autofillTypes = listOf(AutofillType.EmailAddress),
  onFill = { /* update the email */ }
)
TextField(
  modifier = Modifier.onGloballyPositioned {
    autofillNode.boundingBox = it.boundsInWindow()
  }
)
```

Once we have our `AutofillNode` we need to add it to the [`AutofillTree`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillTree.html), which we can access via [`LocalAutofillTree`](https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/package-summary#localautofilltree).

```kotlin
LocalAutofillTree.current += autofillNode
```

The last thing we need is to actually trigger the autofill action, which we can do by grabbing the local [`Autofill`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/Autofill.html) and calling `requestAutofillForNode(AutofillNode)`. Usually you will want to do this when your component receives focus by using [`Modifier.onFocusChanged()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/focus/package-summary#onfocuschanged).

A complete Composable that supports autofill for an email address looks like this:

```kotlin
@Composable
fun AutofillEmail() {
  var email by remember { mutableStateOf("") }
  val autofillNode = AutofillNode(
    autofillTypes = listOf(AutofillType.EmailAddress),
    onFill = { email = it }
  )
  val autofill = LocalAutofill.current

  LocalAutofillTree.current += autofillNode

  OutlinedTextField(
    modifier = Modifier.onGloballyPositioned {
      autofillNode.boundingBox = it.boundsInWindow()
    }.onFocusChanged { focusState ->
      autofill?.run {
        if (focusState.isFocused) {
          requestAutofillForNode(autofillNode)
        } else {
          cancelAutofillForNode(autofillNode)
        }
      }
    },
    value = email,
    onValueChange = { email = it }
  )
}
```

Here's what it looks like in action:
![Email text field with autofill suggestions visible](/public/assets/posts/compose_autofill/email_autofill.png){:height="300px"}

One thing I love about Compose's approach is that it makes manipulating the autofilled data extremely easy. You can perform any sort of mapping or formatting of the input data in the `onFill` callback. Way easier than subclassing a View!

### More concise: An autofill Modifier
That's a lot of code to get autofill working for a single Composable. We can make our lives a bit easier by extracting a lot of that behavior.

The [`ExplicitAutofillTypesDemo`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/ui/ui/integration-tests/ui-demos/src/main/java/androidx/compose/ui/demos/autofill/ExplicitAutofillTypesDemo.kt) gives us a good place to start by offering an `Autofill` composable that you can use to wrap any composable. I think we can do better. I've already mentioned that Google is planning on supporting autofill via a Modifier, so let's try to recreate that!

We need to be able to access our `LocalAutofill` and `LocalAutofillTree` in the Modifier, and [`Modifier.composed()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/package-summary#(androidx.compose.ui.Modifier).composed(kotlin.Function1,%20kotlin.Function1)) gives us that ability. 

```kotlin
fun Modifier.autofill(
  autofillTypes: List<AutofillType>,
  onFill: ((String) -> Unit),
) = composed {
  val autofill = LocalAutofill.current
  val autofillNode = AutofillNode(onFill = onFill, autofillTypes = autofillTypes)
  LocalAutofillTree.current += autofillNode

  this.onGloballyPositioned {
    autofillNode.boundingBox = it.boundsInWindow()
  }.onFocusChanged { focusState ->
    autofill?.run {
      if (focusState.isFocused) {
        requestAutofillForNode(autofillNode)
      } else {
        cancelAutofillForNode(autofillNode)
      }
    }
  }
}
```

Now our text field composable is simple and clean:

```kotlin
@Composable
fun AutofillEmail() {
  var email by remember { mutableStateOf("") }

  OutlinedTextField(
    modifier = Modifier.autofill(
      autofillTypes = listOf(AutofillType.EmailAddress),
      onFill = { email = it },
    ),
    value = email,
    onValueChange = { email = it }
  )
}
```

## Parting thoughts
While this modifier works pretty well there are two things our `Autofill` Composable doesn't do yet that we do get with Views.

First, Views typically show a light yellow background after being autofilled:

![Email text field with yellow background](/public/assets/posts/compose_autofill/yellow_box.png){:height="120px"}

This happens automatically for `EditText` View, but doesn't happen automatically for a `TextField` Composable. If you want this behavior, you could easily reproduce it by applying a `Modifier.background()` after an `onFill` callback.

The second thing that our Composable doesn't support yet is the Autofill action in the context menu:

![Text field context menu with Autofill option](/public/assets/posts/compose_autofill/context_menu.png){:height="200px"}

Compose doesn't currently adding options to a text field's context menu, so this isn't possible right now. That support is [mentioned on the issue tracker](https://issuetracker.google.com/issues/180639271) though!