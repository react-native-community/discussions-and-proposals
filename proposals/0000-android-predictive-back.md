---
title: Predictive back gesture support on Android
author:
- Grzegorz Kruk
- Kacper Kafara
date: 2026-11-17
---

# RFC0000: Predictive back gesture support on Android

## Summary

Let Android show its predictive back animation on React Native screens by default.

On Android 16+, for apps that target SDK 36, `ReactActivity` registers a back callback that is always enabled. An enabled callback tells Android "this app handles back itself", so Android never plays its predictive back animation on a React Native screen, and native navigation libraries never receive the gesture.

This RFC proposes that `ReactActivity` registers this callback **disabled**, and that JavaScript asks for back only when it has something to go back to, with a new `BackHandler.setInterceptEnabled(enabled)` method. Native libraries get access to the callback through a new `ReactActivity.getBackPressedCallback()` method. `BackHandler` keeps its API, but on Android 16+ its guarantees are smaller: it receives back events only while interception is on, and native code can still take back from it.

Gesture progress and predictive back animations stay out of React Native core. They belong in community libraries such as `react-native-screens`.

## Basic example

A screen with its own JavaScript back stack asks for back only while it can go back:

```tsx
import {useEffect} from 'react';
import {BackHandler} from 'react-native';

function useAndroidBack(canGoBack: boolean, goBack: () => void) {
  useEffect(() => {
    BackHandler.setInterceptEnabled(canGoBack);
    const subscription = BackHandler.addEventListener('hardwareBackPress', () => {
      goBack();
      return true;
    });
    return () => {
      subscription.remove();
      BackHandler.setInterceptEnabled(false);
    };
  }, [canGoBack, goBack]);
}
```

While `canGoBack` is `true`, back goes to JavaScript and the listener pops the JavaScript stack, as it does today. When `canGoBack` is `false`, Android handles back and plays its animation, for example back to the home screen.

A native navigation library, or a brownfield host app, that needs to own back:

```kotlin
reactActivity.backPressedCallback.isEnabled = false
```

## Motivation

### What predictive back is

On Android, users go back by swiping from the left or right edge of the screen, or by pressing the back button. With predictive back, Android shows a preview of where back leads _while the finger is still on the screen_. The current screen shrinks and slides away, and the previous screen, or the home screen, appears behind it. If the user lifts the finger, back happens. If the user moves the finger back to the edge, back is cancelled.

Android can only show this preview if it knows ahead of time what back will do. So apps no longer decide _after_ the back press. They declare _before_ the gesture whether they want to handle back.

Apps declare this with back callbacks (`OnBackPressedCallback` in AndroidX). Each activity has a list of callbacks, and each callback is enabled or disabled:

- If any callback is enabled, Android gives back to the app and plays no system animation.
- If all callbacks are disabled, Android handles back itself and plays its animation.
- If several callbacks are enabled, the one enabled last wins.

### How Android rolled it out

| Android version     | Change                                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Android 13 (API 33) | Predictive back added as an opt-in (`android:enableOnBackInvokedCallback="true"`). The back-to-home animation is behind a developer option.                                                                                                                                                                                                                    |
| Android 14 (API 34) | Apps can animate their own screens during the gesture (`OnBackAnimationCallback`). System animations are still behind the developer option.                                                                                                                                                                                                                    |
| Android 15 (API 35) | System animations (back to home, cross-activity, cross-task) are shown to opted-in apps without the developer option.                                                                                                                                                                                                                                          |
| Android 16 (API 36) | On by default for apps that target SDK 36. `Activity.onBackPressed()` is no longer called and `KEYCODE_BACK` is no longer dispatched. Apps can still opt out with `android:enableOnBackInvokedCallback="false"`, which [Android calls temporary](https://developer.android.com/about/versions/16/behavior-changes-16). Adds `PRIORITY_SYSTEM_NAVIGATION_OBSERVER`. |
| Android 17 (API 37) | No change to predictive back. No date has been announced for removing the opt-out.                                                                                                                                                                                                                                                                             |

Google Play raises the minimum target SDK every year, so every React Native app will target SDK 36 and later.

References: [Add support for the predictive back gesture](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture), [`OnBackInvokedDispatcher`](https://developer.android.com/reference/android/window/OnBackInvokedDispatcher).

### Current state in React Native

React Native's back API is `BackHandler`. Each listener gets the back event and returns `true` to say "I handled it" or `false` to let the default happen ([`BackHandler.android.js`](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/Libraries/Utilities/BackHandler.android.js#L23-L37)). This model needs React Native to receive _every_ back event first and decide afterwards.

On Android 16 with target SDK 36, `onBackPressed()` is no longer called, so React Native would stop receiving back. To keep `BackHandler` working, [react-native#51096](https://github.com/react/react-native/pull/51096) added an `OnBackPressedCallback` to `ReactActivity` that is [always enabled](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java#L29-L39) and [registered in `onCreate`](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java#L63-L65). [react-native#54958](https://github.com/react/react-native/pull/54958) made [`invokeDefaultOnBackPressed`](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java#L122-L131) enable it again after each default back action.

This keeps `BackHandler` working, but it has a cost: Android never shows the predictive back animation on a React Native screen. React Native apps are quietly opted out of predictive back, while native apps get it. This affects apps that never use `BackHandler` too.

### Native navigation libraries

`react-native-screens` uses Android fragments for native stacks. Android can animate a fragment pop during the gesture only when the fragment system receives the gesture. The always-enabled `ReactActivity` callback takes the gesture first. `ReactActivity` gives no access to the callback, so the screens example app disables it with reflection ([react-native-screens#3630](https://github.com/software-mansion/react-native-screens/pull/3630)). [react-native#55211](https://github.com/react/react-native/pull/55211) proposes a getter, and has had no review since January 2026.

### Brownfield apps

In a brownfield app, React Native is one part of a larger native app. Native screens and libraries register their own callbacks on the same activity, and the callback enabled last wins. React Native cannot know when another callback takes back from it. Also, all React Native surfaces on one JavaScript runtime share the same `BackHandler` listeners, so a back event meant for one surface reaches the others.

[`ReactFragment`](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactFragment.kt#L115) registers no callback. The host activity must call `ReactFragment.onBackPressed()` itself. Libraries that host React Native in a fragment may register their own always-enabled callback, which blocks the system animation on that screen too.

### Demo apps

Two demo apps show this behavior on an Android 17 emulator, with target SDK 36 and gesture navigation:

- [**rn-predictive-back**](https://github.com/grzegorzkruk/rn-predictive-back): a React Native app with react-navigation, react-native-screens and Reanimated.
- [**rn-predictive-back-brownfield**](https://github.com/grzegorzkruk/rn-predictive-back-brownfield): a native Android app that embeds React Native with react-native-brownfield. The path is native screen → React Native screen (`ReactActivity`) → native screen with an embedded React Native fragment. The [research reports](https://github.com/grzegorzkruk/rn-predictive-back-brownfield/tree/main/research) contain the full results.

In both apps, Android shows the predictive back animation only while the React Native callback is disabled.

### Expected outcome

- Every React Native app gets the system predictive back animation by default.
- Native navigation libraries and brownfield hosts can own back without reflection.
- React Native's back handling follows Android's model, so Android documentation and patterns apply to React Native apps.

## Detailed design

A draft implementation is in [react-native#59292](https://github.com/react/react-native/pull/59292).

### `ReactActivity`

On Android 16+ with target SDK 36+ ([`AndroidVersion.isAtLeastTargetSdk36`](https://github.com/react/react-native/blob/61448351cc2d31d444354e46ef2fb338981aa109/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/util/AndroidVersion.kt#L58-L60) checks both), `ReactActivity` keeps registering `mBackPressedCallback` on the activity's `OnBackPressedDispatcher`, but creates it **disabled**:

```java
private final OnBackPressedCallback mBackPressedCallback =
    new OnBackPressedCallback(false) {
      @Override
      public void handleOnBackPressed() {
        setEnabled(false);
        onBackPressed();
        setEnabled(true);
      }
    };
```

When the callback is enabled and back happens, the existing path runs: `ReactActivityDelegate.onBackPressed()` emits `hardwareBackPress` to JavaScript. If no listener returns `true`, JavaScript calls `invokeDefaultOnBackPressed()`.

`invokeDefaultOnBackPressed()` restores the callback's previous enabled state instead of always enabling it:

```java
@Override
public void invokeDefaultOnBackPressed() {
  boolean enabled = mBackPressedCallback.isEnabled();
  mBackPressedCallback.setEnabled(false);
  super.onBackPressed();
  mBackPressedCallback.setEnabled(enabled);
}
```

Without this, `BackHandler.exitApp()` would turn interception on, and would enable again a callback that a navigation library disabled.

Two new public methods:

```java
/** The callback React Native registers on Android 16+ (target SDK 36+). Starts disabled. */
public OnBackPressedCallback getBackPressedCallback();

/** Called from JavaScript through BackHandler.setInterceptEnabled. */
public void setInterceptEnabled(boolean enabled);
```

On Android below 16, or with a target SDK below 36, the callback is never registered. `Activity.onBackPressed()` is called as before, and `BackHandler` works exactly as today. Calling `setInterceptEnabled` or changing the callback there has no effect.

### JavaScript API

```ts
type BackHandler = {
  // ...existing methods
  setInterceptEnabled(enabled: boolean): void;
};
```

- On Android, it calls a new `setInterceptEnabled(enabled: boolean)` method on the existing `NativeDeviceEventManager` TurboModule spec. `DeviceEventManagerModule` then calls `ReactActivity.setInterceptEnabled(enabled)` on the UI thread.
- On iOS, it does nothing.
- `addEventListener('hardwareBackPress', ...)` does not change. On Android 16+ with target SDK 36+, listeners run only while interception is on.

`setInterceptEnabled` is a **preference, not a guarantee**. Callbacks are dispatched by registration and enable order, so a native callback enabled later still receives back first. JavaScript cannot ask Android whether it still owns back.

### Which activity receives the call

In the draft implementation, `DeviceEventManagerModule` calls `setInterceptEnabled` on `currentActivity`, if it is a `ReactActivity`. This works in apps with one `ReactActivity`. In brownfield apps, the activity in front is often a native activity. Then the call is dropped, and the `ReactActivity` behind it keeps its previous state. In the brownfield demo, a React Native screen kept blocking the system animation after JavaScript had turned interception off.

Possible solutions:

1. Store the requested value per React Native surface or per `ReactHost`, and apply it when a `ReactActivity` resumes.
2. Pass the root tag of the calling surface, so native code can find the correct activity.
3. Keep `currentActivity`, and document that the call targets the React Native activity in front and must be repeated when a screen gets focus.

The first two are preferred. The third is the minimum for a first version.

### Fragments and brownfield apps

React Native core keeps no callback in `ReactFragment`. The host app owns back. It decides when to take the gesture and forwards the final back press to the fragment with `ReactFragment.onBackPressed()`. A host-facing way to ask whether JavaScript wants back is an open question.

Libraries that host React Native in a fragment, such as react-native-brownfield, need to expose or disable their own callback. That is a change in those libraries, not in core.

### Optional: an observe-only "back happened" event

Android 16 adds [`PRIORITY_SYSTEM_NAVIGATION_OBSERVER`](https://developer.android.com/reference/android/window/OnBackInvokedDispatcher#PRIORITY_SYSTEM_NAVIGATION_OBSERVER). An observer callback is called only when no other callback took back, does not block the system animation, and cannot cancel back.

React Native could register one observer and emit a new JavaScript event, for example `backCommitted`, so apps can log back or clean up state. This event must not be `hardwareBackPress`: Android has already acted when the observer runs, often the activity is already finishing, and returning `true` would mean nothing. In the brownfield demo, a prototype that reused `hardwareBackPress` here popped a JavaScript stack on an activity that was already closing.

On Android 16, only one observer can be registered per window, so a React Native observer would stop the host app from registering its own. This can be a follow-up and is not part of the first version.

### Out of scope: gesture progress and animations

Gesture progress (0 to 1), components that animate with the finger, and in-app predictive back animations stay out of core. A prototype added a `PredictiveBackAnimatedView` component and native progress listener APIs to `ReactActivity`. It worked in a plain `ReactActivity`, but:

- it did not work inside a `ReactFragment`, because the host is not a `ReactActivity`;
- it used its own "far enough to go back" rule, which can differ from Android's;
- it added a large public API.

`react-native-screens` gets progress from the fragment system. Other libraries can register their own `OnBackPressedCallback` and use the AndroidX `handleOnBackStarted`, `handleOnBackProgressed` and `handleOnBackCancelled` methods.

## Drawbacks

- **Breaking change on Android 16+ with target SDK 36.** Code that blocks back with `BackHandler` stops receiving back until it calls `setInterceptEnabled(true)`. Examples: JavaScript-only navigators, modals, forms with unsaved changes, "press back twice to exit".
- **Navigation libraries must update.** React Navigation's JavaScript stack, modals and drawers need to call `setInterceptEnabled(canGoBack)`. Until they do, back in those apps closes the activity instead of going back inside JavaScript.
- **No ownership guarantee.** A native callback enabled later wins. JavaScript cannot find out that it lost back, so a screen may believe it blocks back when it does not.
- **Different behavior across Android versions.** Below Android 16, `BackHandler` receives every back event without opting in. Developers who test only on older devices may miss the opt-in.
- **More API surface.** One JavaScript method, one native module method and two public `ReactActivity` methods.

## Alternatives

### Keep the always-enabled callback

`BackHandler` keeps working without changes, but React Native apps never get the system animation, and native navigation libraries keep using reflection. When Google removes the manifest opt-out, React Native will still block the animation.

### Observer only, never block

React Native would never enable a callback. It would only register a `PRIORITY_SYSTEM_NAVIGATION_OBSERVER` and report back events to JavaScript.

- JavaScript loses the ability to prevent the default action, which is the main use of `BackHandler` today.
- The observer runs after Android has acted, often after the activity is closing.
- It works only on Android 16+, allows one observer per window on Android 16, and gets no gesture progress.

It is useful as an extra notification (see the optional event above), but it cannot replace `BackHandler`. Apps that need to block back must still declare it ahead of time.

### Enable the callback automatically while `hardwareBackPress` listeners exist

React Navigation and many libraries always register a listener, so back would stay blocked in almost every app, the same as today.

### Recommend the manifest opt-out

Android calls the opt-out temporary, and it disables all predictive back animations.

### Full predictive back support in core

A prototype added gesture progress events, `PredictiveBackAnimatedView` and an in-app back animation to core. See "Out of scope" above for why it is better placed in community libraries.

### Always win the gesture

React Native could try to always take back first, for example with `PRIORITY_OVERLAY` or a proxy callback. This works against Android's model, breaks native navigation libraries and brownfield hosts, and was rejected.

### Prior art

- **Android:** `OnBackPressedCallback.isEnabled` means "I will handle back" and is set ahead of time. `setInterceptEnabled` maps directly to it.
- **Jetpack Compose:** `BackHandler(enabled = ...)` is the same switch.
- **Flutter:** for Android predictive back, Flutter deprecated `WillPopScope`, which decided after the back press, in favor of `PopScope(canPop: ...)`, which declares ahead of time. Flutter keeps "who handles back" and "who receives progress" as separate mechanisms, and with nested navigators they can pick different owners ([flutter#152578](https://github.com/flutter/flutter/issues/152578)). This is one more reason to keep progress out of core for now.

## Adoption strategy

- Ship the new default behind a feature flag for one release, so apps have time to migrate. Then make it the default for target SDK 36+.
- Changelog: `[ANDROID] [BREAKING]`.
- Coordinate with:
  - **React Navigation and Expo Router:** call `setInterceptEnabled(canGoBack)` in JavaScript stacks, modals and drawers.
  - **react-native-screens:** use `getBackPressedCallback()` instead of reflection when native navigation owns back.
  - **react-native-brownfield:** expose or disable its fragment callback, and document that the host owns back.
- App migration:
  - Apps that never block back: no change. They get the system animation.
  - Apps that block back: call `setInterceptEnabled(true)` only while needed, and turn it off afterwards.

A codemod is not practical, because the correct value of `setInterceptEnabled` depends on app state.

## How we teach this

- **Terminology:** "intercept". Interception on means React Native takes back from Android. Interception off means Android handles back and plays its animation.
- **Key message:** on Android 16+, Android animates back by default. Turn interception on only while your screen has something to go back to. It is a request, not a guarantee.
- **Documentation:**
  - Update the [`BackHandler`](https://reactnative.dev/docs/backhandler) page with `setInterceptEnabled`, the Android 16 behavior and the example above.
  - Add a brownfield section: the host app owns back and forwards it to React Native.
  - Add a migration note to the release blog post.
- **For existing developers:** present it as a continuation of `BackHandler` with one new rule: say ahead of time when you need back.

## Unresolved questions

- Do we accept the breaking change of a disabled default, or is a softer path needed?
- How should `setInterceptEnabled` find the correct activity: per surface or per `ReactHost`, a root tag, or "the React Native activity in front"?
- Naming: `setInterceptEnabled` describes the mechanism. Alternatives: `setBackHandlerEnabled`, `requestBackHandling`.
- Should there be a dev-mode warning when a `hardwareBackPress` listener exists while interception is off?
- Should core add the observe-only event? What should it be called, and how do we handle the one-observer-per-window limit on Android 16?
- Should `ReactDelegate` or `ReactFragment` get a host-facing API, for example "does JavaScript want back?"
- How can JavaScript learn that native code took back from it?
- `BackHandler` listeners are shared by all React Native surfaces on one runtime. Is this in scope, or a separate RFC?
