---
layout: post
title: View Binding&#58; Includes with Conflicting IDs
redirect_from:
  - /2020/03/30/view-binding-merge-include-copy/
---

[View Binding](https://developer.android.com/topic/libraries/view-binding) is a fantastic new tool for Android Developers to interact with their layouts. As a refresher, View Binding helps replace `findViewById()` calls, KotlinX synthetic view accessors, and Butterknife with a solution that is null-safe, View type-safe, and directly tied to your layout files. View Binding was actually split off from the Databinding plugin, which has been stable and used in production apps for a couple years now.

Many applications use `<include>` tags in their layout files and as developers start migrating to View Binding they will need to watch out for conflicting View IDs in their included layouts.
<!--more-->
When using an `<include>` tag with an ID in your layout View Binding will generate a reference to the binding class of the included layout for you. This gives you and easy way to access the included views with all the type and null safety you get with View Binding.

The most common issue that I've run into in migrating code to View Binding is `<include>` tags where both the `<include>` tag itself and the root of the included layout have IDs set like so:

**home_fragment.xml**
{% highlight xml %}
<include
  layout="@layout/common_progress"
  android:id="@+id/home_progress" />
{% endhighlight %}

**common_progress.xml**
{% highlight xml %}
<ProgressBar
  android:id="@+id/progress_bar" />
{% endhighlight %}

When we inflate **home_fragment.xml**, the ID on the `<include>` tag will overwrite the ID on the root view (the `ProgressBar`) in the included **common_progress.xml** layout, so we end up with a `ProgressBar` with an ID of `R.id.home_progress`. This is true regardless of whether you use View Binding or not.

View Binding will generate both a `HomeFragmentBinding` class and a `CommonProgressBinding` class for two layout files as expected. The include shows up as a reference to a `CommonProgressBinding` field in `HomeFragmentBinding`. However, `CommonProgressBinding` doen't know it is being included, so it is looking for a `ProgressBar` with the ID `R.id.progress_bar`!

Thus if we were to run code using `HomeFragmentBinding`, we would run into a crash that looks like this:

> java.lang.NullPointerException: Missing required view with ID: progress_bar
>        at com.example.CommonProgressBinding.bind(ProgressBarBinding.java:73)

There are three ways you can fix this problem right now:

 1. Remove the ID on the root of the included layout (**common_progress.xml**). You can't remove the ID on the `<include>` tag if you want to access it through View Binding, so removing it on the included layout is the way to go.

 2. Make both IDs match. If both the `<include>` and the root view of the included layout have the same ID, both generated binding classes are looking for the same ID and there's no issue!

 3. Wait for Android Studio 4.0. This issue is fixed in version 4.0 of the Android Gradle Plugin! Instead of calling `findViewById()` for the root View, all View Binding classes simply cast the root View to the appropriate type ([see this change request](https://android.googlesource.com/platform/frameworks/data-binding/+/69294d8ad4a0f76a0ef9a0916b4e0c24f30fc84f)).