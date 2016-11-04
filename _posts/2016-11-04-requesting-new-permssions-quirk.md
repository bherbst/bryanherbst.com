---
layout: post
title: Quirk in Requesting Permissions in Groups
---

[Marshmallow brought runtime permissions](https://developer.android.com/training/permissions/requesting.html)
to the Android world, and by now most apps have started requesting their permissions
at runtime instead of install time (you *have*, right?).

Permissions are grouped into buckets like "location" or "storage" to simplify
the explanation of what a permission does for users. Hopefully this is all old news to you.

However there is an interesting situation that merits a little more explanation.
Let's say your app currently requests `ACCESS_COURSE_LOCATION`. Your requirements
have changed a little bit, and you now need `ACCESS_FINE_LOCATION`. What happens might
not be quite what you expect.

The documentation provides this note:

> Note: Your app still needs to explicitly request every permission it needs,
> even if the user has already granted another permission in the same group.

So we know we need to request the new fine location permission. But what does the user see?

### If the user has not accepted `ACCESS_COURSE_LOCATION`

If a user hasn't accepted the coarse location permission (and hasn't said "don't ask again"),
your app behaves as you would expect it to on a fresh install. You can request the permission
as usual and go on about your day.

### If the user has denied `ACCESS_COURSE_LOCATION`

If your user has denied the coarse location permission and selected the "never
ask again" option, you will not be allowed to ask for either permission. This is
also generally what I would expect- the user clearly doesn't want you using their location,
so you don't get the opportunity to ask again.

### If the user has accepted `ACCESS_COURSE_LOCATION`

Here's where it gets a little more interesting.

 * If your user goes into the app's settings page to look at the permissions, they
will see that the location toggle is already "on.
 * If you check for the fine location permission with `checkSelfPermission()`
 you'll still get `PERMISSION_DENIED`.
 * If you request `ACCESS_FINE_LOCATION` with `requestPermissions()`, you will
 automatically get a callback to `onRequestPermissionsResult()` with the fine
 location permission granted. This is transparent to your user- they will not
 see a dialog.

### Summary

If your app already requests a permission in a particular permission group,
you still need to explicitly request all other permissions in that group that you
want to use. However, if your user has already accepted one of the other permissions
in that group, they will not see another permission request dialog.
