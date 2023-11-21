---
title: Add support for CSS color() function
author:
  - Ryan Linton
  - Yulian Glukhenko
date: 2023-11-20
---

# RFC0000: Add support for CSS color() function

## Summary

React Native does [not currently support](https://github.com/facebook/react-native/issues/41517) wide color gamut color spaces (i.e. display-p3). This proposal discusses adding support for the CSS color() function enabling support for additional wide-gamut color spaces.

## Basic example

```js
StyleSheet.create({
  backgroundColor: "color(display-p3 1 0.5 0)",
  color: "color(display-p3 1 0.5 0 / .5)",
});
```

## Motivation

Since most new devices support wider gamut color spaces, React Native should support them as well. The Display P3 color space has had native support since Android 8.0 and iOS 9.3. The color() function was introduced with [CSS Color Module Level 4](https://drafts.csswg.org/css-color/#color-function), much of which is already implemented in React Native. Also Flutter [recently added support](https://github.com/flutter/flutter/issues/55092) for Display P3 on iOS with plans to follow up with support for Android and then the framework itself.

## Detailed design

### JS Changes

1. Parse the color() function in [normalizeColor](https://github.com/facebook/react-native/blob/63213712125795ac082597dad2716258b90cdcd5/packages/normalize-color/index.js#L235)

```js
const COLOR_SPACE = /display-p3|srgb/

function getMatchers() {
  if (cachedMatchers === undefined) {
    cachedMatchers = {
      color: new RegExp(
        'color(' +
          call(COLOR_SPACE, NUMBER, NUMBER, NUMBER) +
          '|' +
          callWithSlashSeparator(COLOR_SPACE, NUMBER, NUMBER, NUMBER, NUMBER) +
          ')'
      ),
```

2. Include the color space in the return value of [StyleSheet processColor](https://github.com/facebook/react-native/blob/63213712125795ac082597dad2716258b90cdcd5/packages/react-native/Libraries/StyleSheet/processColor.js) and [Animated processColor](https://github.com/facebook/react-native/blob/main/packages/react-native/Libraries/Animated/nodes/AnimatedColor.js)

```js
return { ["display-p3"]: true, r, g, b, a };
```

### iOS Changes

1. Update [RCTConvert](https://github.com/facebook/react-native/blob/781b637db4268ad7f5f3910d99ebb5203467840b/packages/react-native/React/Base/RCTConvert.m#L881) to handle setting new color values for color space. If color space is not included preserve existing behavior.

2. Update [RCTConversions](https://github.com/facebook/react-native/blob/16ad818d21773cdf25156642fae83592352ae534/packages/react-native/React/Fabric/RCTConversions.h#L37) to handle setting new color values for color space. If color space is not included preserve existing behavior.

3. Update [RCTTextPrimitivesConversions](https://github.com/facebook/react-native/blob/ac1cdaa71620d5bb4860237cafb108f6aeae9aef/packages/react-native/ReactCommon/react/renderer/textlayoutmanager/platform/ios/react/renderer/textlayoutmanager/RCTTextPrimitivesConversions.h#L116) to handle setting new color values for color space. If color space is not included preserve existing behavior.

```objc
[UIColor colorWithDisplayP3Red:components.red green:components.green blue:components.blue alpha:components.alpha]
```

### Android Changes

1. Enable the wide color gamut in [ReactActivity](https://github.com/facebook/react-native/blob/7625a502960e6b107e77542ff0d6f40fbf957322/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java#L22) if available.

```java
Window window = getWindow();
boolean isScreenWideColorGamut = window.getDecorView().isScreenWideColorGamut()
if (isScreenWideColorGamut) {
  window.setColorMode(PixelFormat.COLOR_MODE_WIDE_COLOR_GAMUT)
}
```

2. Update [ColorUtil](https://github.com/facebook/react-native/blob/a6964b36294c3bfea09c0cdd65c5d0e3949f2dae/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/views/view/ColorUtil.java#L17) and [ColorPropConverter](https://github.com/facebook/react-native/blob/781b637db4268ad7f5f3910d99ebb5203467840b/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/bridge/ColorPropConverter.java#L27) to return new color values for color space as longs.

```java
@ColorLong long p3 = pack(1.0f, 1.0f, 0.0f, 1.0f, ColorSpace.Named.DISPLAY_P3);
Color opaqueYellow = Color.valueOf(p3);
```

## Drawbacks

There should be no breaking changes for users but the color implementation will be a bit more complicated than before.

## Alternatives

We could do nothing and require users to patch package for Display P3 color support. But this is a disappointing developer experience and not a satisfying solution.

## Adoption strategy

This is intended to be a non-breaking change with zero impact to existing users. Adoption is entirely opt-in by using the color function in styles to start using display-p3 colors.

## How we teach this

We should document the inclusion of the color function in the [official color reference](https://reactnative.dev/docs/colors). Developers can then start using it if they choose.

## Unresolved questions

- While Android does support wide color gamut, do [Android View](<https://developer.android.com/reference/android/view/View#setBackgroundColor(int)>)'s actually support it? These methods take `int`s while the Android color reference states wide gamut colors are `long`s.
- Do we need to do additional work to support interpolating colors for animations?
