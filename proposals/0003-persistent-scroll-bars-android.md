---
title: Expose Android View's setScrollbarFadingEnabled via a boolean prop
author:
- K. P. Sroka
date: 2018-10-10
---

# RFC0003: Expose Android View's setScrollbarFadingEnabled via a boolean prop

## Summary

This change aims to expose a method from Android's View API: [setScrollbarFadingEnabled](https://developer.android.com/reference/android/view/View.html#setScrollbarFadingEnabled(boolean)). This method would be exposed as a boolean prop in RN's ScrollView component, implemented through [ReactScrollViewManager](https://github.com/facebook/react-native/blob/1151c096dab17e5d9a6ac05b61aacecd4305f3db/ReactAndroid/src/main/java/com/facebook/react/views/scroll/ReactScrollViewManager.java)'s and [ReactHorizontalScrollViewManager](https://github.com/facebook/react-native/blob/1151c096dab17e5d9a6ac05b61aacecd4305f3db/ReactAndroid/src/main/java/com/facebook/react/views/scroll/ReactHorizontalScrollViewManager.java)'s methods. It would not have any effect on iOS.

## Basic example

```
<ScrollView persistentScrollbar>
  ...children...
</ScrollView>
```
## Motivation

I'm a part of the team behind the [React Native Paper](https://callstack.github.io/react-native-paper/) library, which aims to implement component set compatible with [Google's Material Design](https://material.io/). Part of the design's spec [states the following](https://material.io/design/components/menus.html#behavior): *If not all menu items are displayed at once, menus can be scrollable. In this state, menus show a persistent scrollbar.*

By default the scroll bars on both Android and iOS plaftorms fade if the user doesn't change the scroll position for a short period of time. As there is a method in Android's View API that allows control over the fading behavior, I'd like to expose it to ease in creating the Menu component for the Paper library.

## Detailed design

I propose that a new boolean prop is added to the React Native's [ScrollView](https://facebook.github.io/react-native/docs/scrollview) component that gets translated to call to [android.view.View#setScrollbarFadingEnabled](https://developer.android.com/reference/android/view/View.html#setScrollbarFadingEnabled(boolean)). This would be implemented as a method in both [ReactScrollViewManager](https://github.com/facebook/react-native/blob/1151c096dab17e5d9a6ac05b61aacecd4305f3db/ReactAndroid/src/main/java/com/facebook/react/views/scroll/ReactScrollViewManager.java) and [ReactHorizontalScrollViewManager](https://github.com/facebook/react-native/blob/1151c096dab17e5d9a6ac05b61aacecd4305f3db/ReactAndroid/src/main/java/com/facebook/react/views/scroll/ReactHorizontalScrollViewManager.java) with the `@ReactProp` annotation. I propose that this prop would be actually an inverse of the boolean passed to the native View param, since the default value of "scrollbar fading" in RN is `true` (as that's what the platform's default to), and overriding the default behavior with simplified notation of `<ScrollView persistentScrollbar />` seems more readable than the alternative `<ScrollView fadeScrollbar={false}>`.

As there is no matching API call in Apple's UIKit ([UiScrollView](https://developer.apple.com/documentation/uikit/uiscrollview)] merely exposes the `[flashScrollIndicators](https://developer.apple.com/documentation/uikit/uiscrollview/1619435-flashscrollindicators)` method, which *could* be used to achieve a similar effect, but requires calling that method continuously with high frequency), this prop would only have an effect on Android devices.

## Drawbacks

- This might be a relatively uncommon prop to use,
- Introducing a boolean param to an API is not very future-proof, especially if the underlying APIs decide to extend that option with a wider range of accepted values (though this might not be very probable in this case).
- It's only supported on one plaftorm.

## Alternatives

- Since React Native doesn't expose this functionality, one could think about implementing a separate library (extending `ReactScrollViewManager` and `ReactHorizontalScrollViewManager`) just to expose it.
- One could add a timer that continuously fires noop scroll events to achieve the same effect, without introducing a new prop.

## Adoption strategy

As this change introduces a previously not supported prop, this would be backwards-compatible with older versions of React Native. No transition plan, nor codemod, is required.

## How we teach this

As this is a minor addition to the ScrollView's interface, adding that feature to the ScrollView's documentation should be enough.

## Unresolved questions

- Naming of the prop: should it be -- as suggested -- `persistentScrollbar`, `fadeScrollbar`, or something else?
