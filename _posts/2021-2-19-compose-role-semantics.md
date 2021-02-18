---
layout: post
title: Role Semantics in Jetpack Compose
---

Jetpack Compose alpha 9 introduced an accessibility change that I'm personally excited about. We can now specify a "role" semantic for our Composables that accessibility services can use to provide more context to users. This context can be important to help a visually impaired user understand how their actions will affect your application for interactable elements. 

A great example is an element that allows the user to select from a list of options. Visually impaired users may not have an obvious way to tell whether that element behaves like a checkbox (multiple options are selectable) or a radio button (only one option is selectable at a time). The new role property is intended to convey that type of information.

<!--more-->

TalkBack has long had this concept of a role for elements. It is why TalkBack reads your button with the text "Click Me" as "Click Me, button" ([source](https://github.com/google/talkback/blob/92eb6dd4461e53fc904052b7fbe9b77ddfbf930a/talkback/src/main/java/screensummary/NodeCharacteristics.java#L277-L278)). However, TalkBack has long determined an element's role [based on its Java class](https://github.com/google/talkback/blob/92eb6dd4461e53fc904052b7fbe9b77ddfbf930a/utils/src/main/java/Role.java#L283-L285).

This has long been an annoyance because it makes it more difficult to get consistent TalkBack behavior for custom views. Yes, we can get the same behavior by adding "button" to the content description of any widget that looks like a button but isn't a `Button`. That's easy to forget though, and if TalkBack changes its behavior we need to change our widget again to match.

Relying on the Java class also doesn't work _at all_ with Jetpack Compose. Composables are not Java classes, and aren't even retained in a tree at runtime like Views are.

The role semantic is here to bridge that gap!

Applying a role to a Composable is similar to using any other semantic property. To tell accessibility services that a Composable is a button, we can set the role like so:

```kotlin
@Composable 
fun CustomButton() {
    Box(
        modifier = Modifier.semantics {
            role = Role.Button
        }
    )
}
```

Many of the existing modifiers that add interaction to a Composable now also accept an optional Role argument. For example if you are using `Modifier.toggleable()` to create an element that behaves like a radio button, you can specify the role like so:

```kotlin
@Composable
fun AlmostRadioButton() {
    Box(
        modifier = Modifier.toggleable(
            role = Role.RadioButton,
            // ...
        )
    )
}
```

For a full list of what Roles are currently available, check out the [`Role` documentation](https://developer.android.com/reference/kotlin/androidx/compose/ui/semantics/Role).

Unfortunately the new role semantic is still limited by TalkBack. TalkBack is still looking for specific View classes, so Compose is simply [mapping roles to View classes](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidComposeViewAccessibilityDelegateCompat.android.kt;l=259-268;drc=f1cab8853ca8a98c99f5c475f15e445965134d60). This is something that could change in the future though and this API opens up the possibility of adding additional roles as well!