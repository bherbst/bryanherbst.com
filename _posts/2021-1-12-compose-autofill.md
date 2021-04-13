---
layout: post
title: Autofill with Jetpack Compose
---
Autofill is one of my favorite features new Android from the past few years. I never want to manually type in my address and credit card information again. Autofill makes filling out forms an absolute breeze!

With Jetpack Compose on the horizon I've been seeing a lot of questions around how to support autofill in Jetpack Compose. Yes, autofill is supported in Compose, and here's how you use it!

<!--more-->

_Caveat: Google intends to simplify interaction with the Autofill framework by supporting an autofill `Modifier`, so expect autofill interaction to change in the future. Since Compose is currently in beta with a goal of maintaining API stability, it is unlikely to change prior to a stable 1.0 release._

## Refresher on Autofill
Android provides two main integration points for autofill: **autofill services** are apps such as Dashlane or LastPass that store a user's information and offer to inject that information when autofill is triggered in a nother app. **Autofill clients** are apps that contain fields that need to be filled out.

Autofill clients provide hints to the autofill service that describe what kind of data they are expecting in each field. A log in screen likely exposes fields with the `"username"` and `"password"` autofill hints.

These hints are completely free-form text, but you should generally stick to the constants defined in the [androidx.autofill](https://developer.android.com/reference/androidx/autofill/HintConstants) library to increase the chances that an autofill service supports those types.

### Autofill in Views
Autofill for most Views is pretty straightforward- you just specify an [`android:autofillHint`](https://developer.android.com/reference/androidx/autofill/HintConstants#attr_android:autofillHint) attribute in your XML file or call [`setAutofillHint(String[])`](https://developer.android.com/reference/android/view/View.html#setAutofillHints(java.lang.String...)) in code. 

This works well for common form Views such as `EditText`, but if you want to support it for other View types you may need to create a custom View and override the [`autofill()`](https://developer.android.com/reference/android/view/View#autofill(android.util.SparseArray%3Candroid.view.autofill.AutofillValue%3E)) method to properly handle the incoming autofilled data.

## Autofill in Compose
The first thing we need to add autofill support to a Composable is an [`AutofillNode`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillNode).

An `AutofillNode` needs three things:
 * A list of [`AutofillType`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillType)s it is interested in
 * An `onFill` callback that will be called when the user selects an autofill suggestion
 * A `Rect` with the coordinates of the Composable on screen

Note that `AutofillType` is an enum, not a string like it is in the View framework. I see this as a great improvement to ensure everyone is using a consistent set of autofill hints.

The bounding `Rect` is fairly easy to get- we can use [`Modifier.onGloballyPositioned()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/layout/OnGloballyPositionedModifier#ongloballypositioned) to get the Composable's bounds in the window.

Once we have our `AutofillNode` we need to add it to the [`AutofillTree`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/AutofillTree.html), which we can access via [`LocalAutofillTree`](https://developer.android.com/reference/kotlin/androidx/compose/ui/platform/package-summary#localautofilltree).

The last thing we need is to actually trigger the autofill action, which we can do by grabbing the local [`Autofill`](https://developer.android.com/reference/kotlin/androidx/compose/ui/autofill/Autofill.html) and calling `requestAutofillForNode(AutofillNode)`. Usually you will want to do this when your component receives focus by using [`Modifier.onFocusChanged()`](https://developer.android.com/reference/kotlin/androidx/compose/ui/focus/package-summary#onfocuschanged).

A complete Composable that supports autofill an email address would look like this:

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
![Email text field with autofill suggestions visible](/public/assets/posts/compose_autofill/email_autofill.png)

### Making things easier
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

![Email text field with yellow background](/public/assets/posts/compose_autofill/yellow_box.png)

This happens automatically for `EditText` View, but doesn't happen automatically for a `TextField` Composable. If you want this behavior, you could easily reproduce it by applying a `Modifier.background()` after an `onFill` callback.

The second thing that our Composable doesn't support yet is the Autofill action in the context menu:

![Text field context menu with Autofill option](/public/assets/posts/compose_autofill/context_menu.png)

Compose doesn't currently adding options to a text field's context menu, so this isn't possible right now. That support is [mentioned on the issue tracker](https://issuetracker.google.com/issues/180639271) though!