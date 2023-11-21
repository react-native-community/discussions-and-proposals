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

Since most new devices support wider gamut color spaces React Native should support them as well. The Display P3 color space has had native support since Android 8.0 and iOS 9.3. The color() function was introduced with [CSS Color Module Level 4](https://drafts.csswg.org/css-color/#color-function), much of which is already implemented in React Native. Also Flutter [recently added support](https://github.com/flutter/flutter/issues/55092) for Display P3 on iOS with plans to follow up with support for Android and then the framework itself.

## Detailed design

1. Parse color() function in [normalizeColor](https://github.com/facebook/react-native/blob/63213712125795ac082597dad2716258b90cdcd5/packages/normalize-color/index.js#L235)
2. Include the color space info in the return value of [processColor](https://github.com/facebook/react-native/blob/63213712125795ac082597dad2716258b90cdcd5/packages/react-native/Libraries/StyleSheet/processColor.js) (i.e. display-p3)
3. Update iOS to handle new color values (UIColor colorWithDisplayP3Red:green:blue:alpha:)
4. Update Android to handle new color values (@ColorLong long p3 = pack(1.0f, 1.0f, 0.0f, 1.0f, ColorSpace.Named.DISPLAY_P3); Color opaqueYellow = Color.valueOf(p3);)

## Drawbacks

- Color parsing will be slightly more complicated than before.

## Alternatives

We could do nothing and require users to patch package for Display P3 color support. But this is a disappointing developer experience and not a satisfying solution.

## Adoption strategy

This is intended to be a non-breaking change with zero impact to existing users. Adoption is entirely opt-in by using the color function in styles to start using display-p3 colors.

## How we teach this

We should document the inclusion of the color function in the [official color reference](https://reactnative.dev/docs/colors). Developers can then start using it if they choose.

## Unresolved questions

While Android does support wide color gamut do [Android View](<https://developer.android.com/reference/android/view/View#setBackgroundColor(int)>)'s actually support it? These methods take `int`s while the Android color reference states wide gamut colors are `long`s.
