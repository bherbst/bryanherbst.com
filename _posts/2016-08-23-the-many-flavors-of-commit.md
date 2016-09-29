---
layout: post
title: The Many Flavors of Commit()
type: medium
medium_link: https://medium.com/@bherbst/the-many-flavors-of-commit-186608a015b1
---

FragmentTransaction in the support library now provides four different ways to commit a transaction:

 * `commit()`
 * `commitAllowingStateLoss()`
 * `commitNow()`
 * `commitNowAllowingStateLoss()`

You have probably also encountered one of these alongside a call to `executePendingTransactions()`.
What do each of these do, and which one should you be using? Let’s explore each one in more depth and find out!
