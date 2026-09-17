---
title: Migrate to androidx.appcompat.app.AlertDialog
author:
- Peter Abbondanzo
date: 2024-05-08
---

# RFC0000: Migrate to androidx.appcompat.app.AlertDialog

## Summary

This proposal suggests migrating [all existing uses](https://github.com/search?q=repo%3Afacebook%2Freact-native+android.app.AlertDialog&type=code) of `android.app.AlertDialog` to `androidx.appcompat.app.AlertDialog` to add support library features. Currently, the two primary uses are in the `AlertFragment` and `DevSupportManagerBase` classes.

## Motivation

Over the years, the vanilla `android.app.AlertDialog` has presented a suite of issues related [dark mode](https://github.com/facebook/react-native/issues/31345), [sizing](https://github.com/facebook/react-native/issues/25881), [RTL support](https://github.com/facebook/react-native/issues/25948), and [layout](https://github.com/facebook/react-native/pull/29832), just to list a few. It's the [explicit recommendation of Android](https://developer.android.com/topic/libraries/support-library) to use the support library wherever possible.

We already use support library components elsewhere to power components like `ReactTextView`, `ReactSwitch`, and `ReactActivity` itelf. Moreover, we make use of the `AppCompatDelegate` to apply theming to support components in our `AppearanceModule`.

Migrating to `androidx.appcompat.app.AlertDialog` would mitigate most of the aforementioned issues as it implicitly enforces using an `AppCompat` theme and provides compatible dark mode support out of the box without having to ask developers to modify their setup.

## Detailed design

There are two primary changes to make: in `AlertFragment` and `DevSupportManagerBase`.

The `AlertFragment` will have to be modified to use the `androidx.appcompat.app.AlertDialog` builder instead of `android.app.AlertDialog`'s builder. Since the class inherits from `DialogFragment`, it only needs to yield a `Dialog` type.

Similarly, the `DevSupportManagerBase` will also have to make this change from `android.app.AlertDialog` to `androidx.appcompat.app.AlertDialog`.

To support activities that do not have AppCompat support, we will need to perform runtime checks against the current activity. This is necessary to ensure nothing crashes, as `AppCompatDelegateImpl` throws an `IllegalStateException` if the current activity does not have an AppCompat theme.

The check would look something like this:

```kotlin
// ThemeUtils.kt
fun isAppCompatTheme(activityContext: Context): Boolean {
  val typedArray = activityContext.obtainStyledAttributes(R.styleable.AppCompatTheme)
  return typedArray.hasValue(R.styleable.AppCompatTheme_windowActionBar)
}
```

We would rework the static `createDialog` function that `AlertFragment` exposes, performing this check up front and logging warnings to users that fail the check:

```kotlin
fun createDialog(
  activityContext: Context,
  arguments: Bundle,
  fragment: DialogInterface.OnClickListener ,
): Dialog {
  return if (ThemeUtils.isAppCompatTheme(activityContext)) {
    // return androidx.appcompat.app.AlertDialog result
  } else {
    // Warning logs here
    // return android.app.AlertDialog result
  }
}
```

And in `DevSupportManagerBase` we would make the same check with logging. To ensure that users perform this upgrade right away, we should also make this check and perform logging whenever a ReactContext is created from this module.

## Drawbacks

There are a few reasons why we would not want to do this:
- This deprecates support for non-AppCompat activities. We are explicitly choosing to drop support for the `android.app.AlertDialog` and its theming quirks
- Theming is now applied via `alertDialogTheme`, not `android:alertDialogTheme`. This is a breaking change and needs to be validated by users who upgrade to the version that contains this change.

## Alternatives

I considered dropping support for `android.app.AlertDialog` altogether, primarily because `ReactActivity` already subclasses `AppCompatActivity`. However, this would negatively impact brownfield developers.

The only other alternative is to leave this component as-is.

## Adoption strategy

This introduces a subtle breaking change in the form of theme support changes. Theming is now applied via `alertDialogTheme`, not `android:alertDialogTheme`. We would have to include a blurb in the release about the theme changes and dropping support for `android.app.AlertDialog`.

As the change is significant enough, it should go out in the next minor release (for example, we're on 0.74.2 so this would ship with 0.75.0). 

Upon communicating this change, the release docs as well as the logging we introduce should point to this RFC. The vast majority of developers should already be using AppCompat activities so the cohort of impacted users will be minimal.

After we give users ample opportunity to upgrade, we can drop support for `android.app.AlertDialog` in a future update.

## How we teach this

This change reinforces design patterns that have been chosen in other parts of the codebase: leaning on Android's support libraries for maximum compatibility. The `AlertFragment` is using an old component and should be modernized to ensure dark mode, layouts, sizing, and everything else behaves the way developers expect it to.

## Unresolved questions

I'm unsure if dropping support for `android.app.AlertDialog` is a necessary step, or what kind of timeline should be followed to do so. Adding support in the first place adds some complexity so dropping it eventually is worthwhile.
