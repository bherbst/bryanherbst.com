---
layout: post
title: Signing an Android app on a CI server
---

I often seen projects that use some sort of custom Gradle configuration to set up signing their APK/AAB for release. Most of the time this involves a file or custom command line arguments that provide extra Gradle project properties that the app's **build.gradle** references in a `signingConfigs` block. 

The Android Gradle Plugin already has a mechanism for you to provide an alternative signing configuration at run time, so you don't need to set this up yourself!

When you run a Gradle task (e.g. `./gradlew assembleDebug`) you can provide an *injected* signing configuration like so:

{% highlight shell %}
./gradlew assembleRelease\
  -Pandroid.injected.signing.store.file=[path_to_keystore] \
  -Pandroid.injected.signing.store.password=[keystore_password] \
  -Pandroid.injected.signing.store.key.alias=[key_alias] \
  -Pandroid.injected.signing.store.key.password=[key_password]
{% endhighlight %}

If you haven't seen this syntax before, the `-P[property_name]` arguments are Gradle project properties that your build scripts and plugins (like the Android Gradle Plugin!) can read.

The Android Gradle Plugin will apply this injected signing configuration to whatever build (e.g. release or debug) you are running, overriding any signing configuration you have in your build.gradle.

On your CI server, you should ideally store the keystore and passwords in your CI's secure storage mechanism. On Jenkins for example you can store the keystore as a "secret file" credential and the rest in "secret text" credentials.