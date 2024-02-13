---
layout: post
title: Fragment bundle size with Hilt and ViewModel
---
[TransactionTooLargeException](https://developer.android.com/reference/android/os/TransactionTooLargeException) can be a difficult crash to fix on Android. For developers who haven't had the joy of troubleshooting `TransactionTooLargeException` before, here's an overview:

When your Activity is going into the background `onSaveInstanceState()` is your application's opportunity to persist any transient state to a `Bundle`. This is useful for maintaining state that would otherwise be lost, such as text a user has entered
 into a text field. Ultimately this results in a [`Binder`](https://developer.android.com/reference/android/os/Binder) transaction - one of Android's primary tools for inter-process communication (IPC). This allows Android to save your bundled state outside of your process so if your process is killed you can restore that state later.

Binder transactions come with a limitation. The transaction buffer (which is shared across your entire process) [is capped at 1MB](https://developer.android.com/guide/components/activities/parcelables-and-bundles). While the documented cap is 1MB, it appears that in practice the OS will `TransactionTooLargeException` when your application attempts to save more than 500KB.

Beacuse of this limitation, Google suggests saving no more than 50kb of data total. That includes saved view state, anything you explicitly save in `onSaveInstanceState()`, and Fragment arguments.

## ViewModel & SavedStateHandle
ViewModels often contain application state that we may want to persist to our application's saved instance state. AndroidX offers a ["Saved State module for ViewModel" (`lifecycle-viewmodel-savedstate`)](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-savedstate) that gives ViewModels an API for doing just that. Simply add a `SavedStateHandle` argument to your ViewModel's constructor, and the default ViewModel provider will take care of hooking up the state saving and restoring mechanisms.

When using a `ViewModel` with a `Fragment`, that Fragment's arguments are included in the `SavedStateHandle` by default, giving you easy access to fragment arguments in your `ViewModel`.

There's a catch here - your `Fragment` is going to save its arguments already, and the `SavedStateHandle` is going to save all of its contents separately. That means that we are going to end up with a duplicate copy of the Fragment arguments: one in the `Fragment`'s saved state and one in the `SavedStateHandle`'s state.

This generally isn't a problem if you are only including very small amounts of data in your Fragment arguments such as a string or two. 
However if your Fragment is already putting too much data or large objects into the arguments this will multiply the impact of those arguments.

## Hilt - compounding the SavedStateHandle problem
The issue with duplicating `Fragment` arguments can exponentially increase if you use Hilt to inject multiple view models in a single screen. 

[The default `SavedStateViewModelFactory` only creates a `SavedStateHandle` if the `ViewModel` requests one](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-viewmodel-savedstate/src/main/java/androidx/lifecycle/SavedStateViewModelFactory.kt;l=126-129;drc=187e9a2088c3f281cfb617b9bedd94b9a3546d8b). This is exactly what we want - don't bother creating a `SavedStateHandle` if we don't need one.

Hilt unfortunately has to support not just injecting your `ViewModel` constructor directly, but also injecting a `SavedStateHandle` into any of your `ViewModel`'s dependencies. As a result, Hilt will create a `SavedStateHandle` for every `ViewModel` even if are not using it!

What that means is that if your `Fragment` uses three different `ViewModel`s, you end up with the same Fragment arguments `Bundle` saved four times: once in the Fragment and once for each `ViewModel`.

## A workaround
The good news is that you can mitigate this! Before I show you how, keep in mind that the best long-term "fix" here is to ensure your arguments are as small as possible. For sufficiently small arguments, this issue is very minor. But if you know your arguments are already too big this can give you a little bit of breathing room.

[Fragment.getDefaultViewModelCreationExtras()](https://developer.android.com/reference/androidx/fragment/app/Fragment#getDefaultViewModelCreationExtras()) is where Fragment passes its arguments to `SavedStateHandle` ([source here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:fragment/fragment/src/main/java/androidx/fragment/app/Fragment.java;l=506-508;drc=e395f84de0068fa4301d043c1b8ea5b8b2982645)). We can override that method and choose to exclude our Fragment's arguments.

```kotlin
override val defaultViewModelCreationExtras: CreationExtras
    get() {
        // Sadly MutableCreationExtras() doesn't support
        // removing key-value pairs, so we need to selectively
        // copy the pieces we want to keep
        val extras = MutableCreationExtras()

        // These default extras are taken from 
        // Fragment.getDefaultViewModelCreationExtras()
        extras.copyExtra(
          defaultExtras, 
          ViewModelProvider.AndroidViewModelFactory.APPLICATION_KEY
        )
        extras.copyExtra(defaultExtras, SAVED_STATE_REGISTRY_OWNER_KEY)
        extras.copyExtra(defaultExtras, VIEW_MODEL_STORE_OWNER_KEY)

        return extras
    }

/**
 * Copy a CreationExtra from one MutableCreationExtras to another
 */
private fun <T : Any> MutableCreationExtras.copyExtra(
    other: CreationExtras,
    key: CreationExtras.Key<T>
) {
  other.get(key)?.let { value ->
    set(key, value)
  }
}
```

This results in nearly identical `ViewModel` creation, just without the Fragment arguments.

Note - if you are actually using the Fragment arguments in your `ViewModel`, you don't want to do this!

## Wrap-up
This is definitely not something I recommend doing proactively. It can lead to unexpected behavior later down the line if someone wants to use those Fragment arguments in a `ViewModel`. It is also a premature optimization if your application isn't close to the binder transaction limit.

Before going down this path you should inspect your saved state bundles using something like [TooLargeTool](https://github.com/guardian/toolargetool) to see which bundles are problematic. Again, the _best_ fix is really minimizing how much infomration you are putting in your Fragment arguments. Typically you want to keep arguments to simple identifiers, not entire objects.

In many cases that might be a longer-term project for you, and hopefully this trick with your `SavedStateHandle` will help!