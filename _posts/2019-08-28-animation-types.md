---
layout: post
title: Android Animation Tool Guide
---

Android has what can be an intimidating number of ways to animate your UI. Of course we have the classics like `View` property animation, and we have more modern approaches like the `Transition` framework. How do you know which one to use? This post is intented to serve as a high level guide to choosing an animation strategy, not to provide an extensive guide to implementation. You'll find links to help with implementation for most of the animations below.

Here's a guide as to which Animation framework you want for different types of animation.

I want to animate...
* A Simple image or icon
* A More complicated image
* Properties of a single View (size, location, etc.)
* Multiple Views in a layout
* The transition between screens

GIFs to get:
 * Icon
 * Lottie
 * Single view moving, changing color, shape
 * Physics-based
 * Layout animation
 * Screen transition

## Simple image or icon
For simple images or icons you should use an [`AnimatedVectorDrawable`](https://developer.android.com/reference/android/graphics/drawable/AnimatedVectorDrawable.html). These scale very well to different screen sizes and it is easy to tweak the animations without dedicated editing software.

[Shape Shifter](https://shapeshifter.design/)] by Alex Lockwood is a phenomenal web tool for creating `AnimatedVectorDrawable`s if you already have vector art that you want to animate. It is much easier than trying to 
hand-craft your AVDs!

## More complicated image
`AnimatedVectorDrawable` is great for relatively simple images like icons, but can be unwieldy and inefficient for more complicated images with a lot of detail or many moving elements. Lottie (by Airbnb) is an excellent choice if you can use AfterEffects. Otherwise a simple `AnimatedDrawable` may be a good fit.

### Lottie
[Lottie](https://airbnb.io/lottie/#/) is a popular library that works across Android, iOS, and the web for rendering Adobe After Effects animations. If your app has a dedicated designer, they may already have familiarity with After Effects, and Lottie allows them to easily work in the tools they are comfortable instead of trying to communicate the animation requirements for you to implement natively on each platform.

Lottie animations are JSON files that After Effects generates using the Bodymovin extension. Simply put the JSON file in your application, plop a `LottieAnimationView` into your app, and you'll get awesome high quality animations!

### `AnimatedDrawable`
If Lottie isn't an option, an [`AnimatedDrawable`](https://developer.android.com/reference/android/graphics/drawable/AnimationDrawable.html) is your next best bet. This requires exporting each frame of the animation as its own drawable. You then combine each of the frames in a single `<animation-list>` drawable and specify the duration of each frame.

This is a much more manual approach, but works well if you don't have the resources to create animations in After Effects.

## Properties of a single View
`ViewPropertyAnimator`s are the easy choice for animating basic properties of a single View.
If you need to perform an animation that `ViewPropertyAnimator` doesn't support, you'll want to use property animation.
For animations started by user interaction like dragging or swiping a View, you might want to consider physics-based animations to provide a natural feel.

### `android.view.animation`
The oldest animation tools in the Android APIs are those in the `android.view.animation` package, which have been around since Android 1.0. They provide a rudimentary way to define View animations. **You shouldn't use these tools**. There are newer counterparts to these APIs that are smarter and easier to use. Nick Butcher [considers them effectively deprecated](https://www.youtube.com/watch?v=N_x7SV3I3P0).

Window animations do still require you to use these APIs to animate Views, but if that's not what you are doing, don't use these!

### Property Animations
[Property animation](https://developer.android.com/guide/topics/graphics/prop-animation.html) (the `android.animation` package) wasn't added until API 11, but forms the basic building blocks for the majority of animations we use on Android today. Chances are your `minSdkVersion` is higher than 11 (or at least it should be!), so using these shouldn't be a problem.

Using the property animation APIs we can animate just about any View property, An [`ObjectAnimator`](https://developer.android.com/reference/android/animation/ObjectAnimator) takes in an object and can animate any property of that object via reflection. For example, here's how to translate a View along the X-axis:

```
ObjectAnimator.ofFloat(view, "translationX", 100f).apply {
    duration = 2000
    start()
}
```

This `ObjectAnimator` will translate our view from its starting position to a translation of 100 pixels over the course of 2,000 ms (2 seconds).

How the animator gets from 0 to 100 depends on our animation's *interpolator*. An [`AccelerateDecelerateInterpolator`](https://developer.android.com/reference/android/view/animation/AccelerateDecelerateInterpolator.html) for example would start and end slowly (smaller changes in the person's age every frame) but the middle of the animation would happen more quickly (bigger changes in every frame). A [`LinearInterpolator`](https://developer.android.com/reference/android/view/animation/LinearInterpolator) on the other hand would make each step the same size (1 year every 2ms in our example).

Note that when you use `ObjectAnimator` you specify the name of a property as a String. `ObjectAnimator` looks for a getter and setter for this property using reflection. If you provide `"translationX"` as the property name, `ObjectAnimator` will look for methods called `setTranslationX()` and `getTranslationX()`.

If your view doesn't have these methods for the property you need you can use a [`ValueAnimator`](https://developer.android.com/reference/android/animation/ValueAnimator.html) instead. `ValueAnimator` doesn't make any changes to existing objects. It just animates between different values and leaves the rest up to you. You could use one to replicate our translation animation like so:

```
    ValueAnimator.ofFloat(0, 100)
      .setDuration(2000)
      .addUpdateListener { translation -> view.setTranslationX(translation) }
```

You can coordinate multiple animators using an `AnimationSet`(https://developer.android.com/reference/android/view/animation/AnimationSet). 

### View Property Animator
The `ViewPropertyAnimator` APIs are really just a suite of convenience methods over creating proeprty animators for your Views. You start by simply calling [`View.animate()`](https://developer.android.com/reference/android/view/View.html#animate())! This will return a [`ViewPropertAnimator`](https://developer.android.com/reference/android/view/ViewPropertyAnimator.html) that offers a concise and fluid syntax for animating a View. Here's an animation that translates and scales a View:

```
view.animate()
    .scaleX(2f).scaleY(2f)
    .translateXBy(50f).translateYBy(50f)
    .start()
```

In addition to providing a more concise API for animating Views, `ViewPropertyAnimator` can be more efficient that `ObjectAnimator`s for the same properties.

These are great for one-off animations, but they are difficult to coordinate with other animations. You don't have a lot of control of the animation once it is running as well.

### Physics-based Animation
AndroidX recently introduced a [physics-based animation](https://developer.android.com/guide/topics/graphics/spring-animation) library. These are great for providing more natural animation to views that are moving due to user interaction like dragging. 

They can be distracting if you use them for all your animations, but when used sparingly alongside user interaction, they can make your animations feel more life-like.

## Multiple Views
Once we have a single View animating, the next step is to coordinate multiple Views in a layout animating together.

For animations on Views that the user is interacting with (e.g. clicking, dragging), look no farther than `MotionLayout`.
For relatively simple layout animations, `TransitionManager.beginDelayedTransition()` should be your go-to.

### AnimationSet
If you have just a few animations that you want to play together, an [`AnimationSet`](https://developer.android.com/reference/android/view/animation/AnimationSet) is a fine option for basic coordination. 

However, the set up for both the individual animators and the `AnimationSet` can get a bit unwieldy. If you are animating more than just one or two properties on one or two Views, I recommend exploring other options.

### Layout animations
There is an older and less used API for animating layout changes such as adding or removing a View known as [`LayoutTransition`](https://developer.android.com/reference/android/animation/LayoutTransition.html). For layouts such as LinearLayout with build in support for these types of animations, you simply need to add `android:animateLayoutChanges="true"` in your layout XML and then changing visibility and adding or removing views will automatically animate!

Unfortunately many ViewGroups don't have built in support for `LayoutTransition`, and the API isn't as easy to use as `MotionLayout` and `Transition`s are. If your layout doesn't already support it or doesn't quite work the way you want, I recommend skipping it. However, they can be an easy way to get basic animations for simple layouts.

### Transitions
Android provides a robust [transition framework](https://developer.android.com/training/transitions) for animating between different "scenes" for a layout or group of Views. A `Scene` is simply a snapshot of the state of your views, and a `Transition` defines how to get from one `Scene` to another.

The `Transition` framework is great because you don't need to calculate the start and end states for all the Views you want to animate, which you need to do if you were to perform the same operations with the basic View animation tooling.

`Transition`s operate on ViewGroups, and have APIs for either including or excluding specific Views by ID, type, or reference. By default all Views in the ViewGroup are included in the Transition.

Android provides a handful of Transitions to handle the most common use cases:
* `Fade` for fading Views in and out
* `ChangeBounds` for moving and resizing Views
* `AutoTransition` for combining `Fade` and `ChangeBounds`. 

### TransitionManager.beginDelayedTransition()
[`TransitionManager.beginDelayedTransition()`] is probably my favorite Android API. It auto-magically creates and starts a Transition for you!

The process looks a bit like this:
1. Call `TransitionManager.beginDelayedTransition(viewGroup)`, passing in the root ViewGroup for the Views you want to animate. The current layout state will be the beginning `Scene` for the transition.
1. Make any changes you want to your layout- show and hide views, move them around, change their sizes, etc.
1. On the next layout pass, Android will create another scene with all your new changes, and start a transition between the two scenes.

### MotionLayout
[`MotionLayout`](https://developer.android.com/training/constraint-layout/motion-layout) is an AndroidX library that offers incredibly powerful animations driven by user interaction. Think interactions like opening a navigation drawer, clicking a button, or scrolling content. One of MotionLayout's best features is that it is built on `ConstraintLayout`, so you get all of that powerful layout tooling.

`MotionLayout` is fully declarative and is unique in that it supports seekable animations. The advantage to supporting seekable animations is that you can use it to drive complex animations that follow user interaction such as dragging across the screen.

One downside to `MotionLayout` is that it only supports animating its direct children, unlike the Transition framework which works with arbitrarily nested children.

Google also plans to add a motion editor in Android Studio that supports previewing and tweaking `MotionLayout` animations.

For more on `MotionLayout`, I highly recommend Google's [Introduction to MotionLayout](https://medium.com/google-developers/introduction-to-motionlayout-part-i-29208674b10d) Medium series.

## Screen transitions
Screen transition animations can be the most difficult animations to get working correctly because they by nature are working across different view hierarchies as new layouts are inflated and destroyed.

Unfortunately there aren't a ton of options here- you really only have one option based on what level of framework component you are working with (Window, Activity, or Fragment).

These animations do share some common terminology though. Let's say we have two Fragments: `HomeFragment` and `DetailFragment`. When we transition from `HomeFragment` to `DetailFragment`, HomeFragment does an **exit** transition and DetailFragment does an **exit** transition. If the user hits the back button, the DetailFragment does a **return** transition and the HomeFragment does a **reenter** transition.

### Window and Activity transition animations
Window content transitions play when moving between `Windows`. For most applications, this means when switching between Activities.
You can define these transitions either in your theme (which will apply the transition to all Activities using that theme), or you can directly set a transition on your Window in code.

These transitions use the same `Transition` framework as described above for layout transitions.

For more, see the [Start an activity using animation](https://developer.android.com/training/transitions/start-activity#start-transition) documentation.

### Fragments
Adding transitions to Fragment transactions is easy! When performing a Fragment transaction, you can call [`setEnterTransition()`](http://developer.android.com/reference/android/support/v4/app/Fragment.html#setEnterTransition%28java.lang.Object%29), [`setExitTransition()`](http://developer.android.com/reference/android/support/v4/app/Fragment.html#setExitTransition%28java.lang.Object%29), [`setReturnTransition()`](http://developer.android.com/reference/android/support/v4/app/Fragment.html#setReturnTransition%28java.lang.Object%29), and [`setReenterTransition()`]http://developer.android.com/reference/android/support/v4/app/Fragment.html#setReenterTransition%28java.lang.Object%29) on your `Fragment` objects. That's it!

For a complete example for Fragment transitions, including some of the commong pitfalls you may run into, I recommend Chris Banes' [Fragment Transitions](https://medium.com/androiddevelopers/fragment-transitions-ea2726c3f36f) Medium article.

### Shared Elements
Often when we transition from one screen to another one or more elements exist on both screens. For example, when going from a list of products on a website to a product details screen, the product image is generally consistent. We can create some pretty awesome animations that take advantage of these "shared elements" to make the transition feel more cohesive.

When launching an Activity, you just need to add an `options` Bundle to your `startActivity()` call to specify which elements are shared. Mike Scamell has a great guide to using Activity shared element transitions [here](https://mikescamell.com/shared-element-transitions-part-1/).

If you are transitioning between Fragments, you can call `addSharedElement()` to your `FragmentTransaction` instead. For a complete guide to shared element transitions with Fragments, check out my [Fragment Transitions with Shared Elements](https://medium.com/@bherbst/fragment-transitions-with-shared-elements-7c7d71d31cbb) article on Medium.

The key for both Activities and Fragments is that each shared element needs a `transitionName` defined in the second screen. This is how the transition framework knows how to find your view in the new screen once the layout is inflated. Think about it like a View ID but just for transitions!