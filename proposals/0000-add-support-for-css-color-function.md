---
title: Add Support for Wide Gamut (DisplayP3) Colors to React Native
author:
  - Ryan Linton
  - Yulian Glukhenko
date: 2023-11-20
---

# RFC0000: Add Support for Wide Gamut (DisplayP3) Colors to React Native

## Summary

React Native does [not currently support](https://github.com/facebook/react-native/issues/41517) wide gamut color spaces (i.e. display-p3). This proposal discusses adding support for this color space in React Native, covering:

- JS implementation
- platform changes

## Basic example

Add a DisplayP3 background color and text color using color() function syntax per the [W3C CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/#color-function) spec.

### Using StyleSheet

```js
StyleSheet.create({
  view: { backgroundColor: "color(display-p3 1 0.5 0)" },
  text: { color: "color(display-p3 0 0.5 1)" },
});
```

### Using inline styles

```jsx
const MyComp = () => (
  <View style={{ backgroundColor: "color(display-p3 1 0.5 0)" }}>
    <Text style={{ color: "color(display-p3 0 0.5 1)"  }}>
  </View>
)
```

## Motivation

Since most new devices support wider gamut color spaces, React Native should support them as well. The DisplayP3 color space has had native support since Android 8.0 and iOS 9.3. The color() function was introduced with [CSS Color Module Level 4](https://www.w3.org/TR/css-color-4/#color-function), much of which is already implemented in React Native. Flutter also [recently added support](https://github.com/flutter/flutter/issues/55092) for DisplayP3 on iOS with plans to follow up with support for Android and then the framework itself. Support for DisplayP3 colors can improve the quality of React Native apps by providing developers a wider range of available colors to use and is aligned with React Native's mission to provide the same look and feel as the underlying native platforms.

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
return { ["display-p3"]: true, red, green, blue, alpha };
```

### iOS Changes

1. Update [RCTConvert](https://github.com/facebook/react-native/blob/781b637db4268ad7f5f3910d99ebb5203467840b/packages/react-native/React/Base/RCTConvert.m#L881) to handle setting new color values for color space. If color space is not included preserve existing behavior.

```objc
+ (UIColor *)UIColor:(id)json
{
  if (!json) {
    return nil;
  }
  if ([json isKindOfClass:[NSArray class]]) {
    NSArray *components = [self NSNumberArray:json];
    CGFloat alpha = components.count > 3 ? [self CGFloat:components[3]] : 1.0;
    return [UIColor colorWithRed:[self CGFloat:components[0]]
                           green:[self CGFloat:components[1]]
                            blue:[self CGFloat:components[2]]
                           alpha:alpha];
  } else if ([json isKindOfClass:[NSNumber class]]) {
    // ...
  } else if ([json isKindOfClass:[NSDictionary class]]) {
    NSDictionary *dictionary = json;
    id value = nil;

    // handle json with srgb color space specified
    if ((value = [dictionary objectForKey:@"srgb"])) {
      return [RCTConvert @[value.red, value.green, value.blue, value.alpha]];

    // handle json with display-p3 color space specified
    } else if ((value = [dictionary objectForKey:@"display-p3"])) {
      return [UIColor colorWithDisplayP3Red:value.red green:value.green blue:value.blue alpha:value.alpha];

    // ...
```

2. Update [ColorComponents.h](https://github.com/facebook/react-native/blob/528f97152b7e0a7465c5b5c02e96c2c4306c78fe/packages/react-native/ReactCommon/react/renderer/graphics/ColorComponents.h) to include color space.

```cpp
enum class ColorSpace {
  sRGB,
  DisplayP3
};

struct ColorComponents {
  float red{0};
  float green{0};
  float blue{0};
  float alpha{0};
  ColorSpace colorSpace{ColorSpace::sRGB};  // Default to sRGB
};
```

3. Update [RCTConversions](https://github.com/facebook/react-native/blob/16ad818d21773cdf25156642fae83592352ae534/packages/react-native/React/Fabric/RCTConversions.h#L37) to handle setting new color values for color space. If color space is not included preserve existing behavior.

4. Update [RCTTextPrimitivesConversions](https://github.com/facebook/react-native/blob/ac1cdaa71620d5bb4860237cafb108f6aeae9aef/packages/react-native/ReactCommon/react/renderer/textlayoutmanager/platform/ios/react/renderer/textlayoutmanager/RCTTextPrimitivesConversions.h#L116) to handle setting new color values for color space. If color space is not included preserve existing behavior.

```cpp
auto components = facebook::react::colorComponentsFromColor(sharedColor);
if (components.colorSpace == ColorSpace::DisplayP3) {
  return [UIColor colorWithDisplayP3Red:components.red green:components.green blue:components.blue alpha:components.alpha];
} else {
  return [UIColor colorWithRed:components.red green:components.green blue:components.blue alpha:components.alpha];
}
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

This is prerequisite per [Android documentation](https://developer.android.com/training/wide-color-gamut) but they note:

> When wide color gamut mode is enabled, the activity's window uses more memory and GPU processing for screen composition. Before enabling wide color gamut mode, you should carefully consider if the activity truly benefits from it. For example, an activity that displays photos in fullscreen is a good candidate for wide color gamut mode, but an activity that shows small thumbnails is not.

So we'll likely want to provide a way to disable this as well.

This won't impact any existing 32-bit integer colors since those will continue to be in sRGB color space. To actually use DisplayP3 colors it is necessary to pack the color space and 4 color components into a 64-bit long value.

2. Update [ColorUtil](https://github.com/facebook/react-native/blob/a6964b36294c3bfea09c0cdd65c5d0e3949f2dae/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/views/view/ColorUtil.java#L17) and [ColorPropConverter](https://github.com/facebook/react-native/blob/781b637db4268ad7f5f3910d99ebb5203467840b/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/bridge/ColorPropConverter.java#L27) to return new color values for color space as longs.

```java
if (value instanceof ReadableMap) {
  ReadableMap map = (ReadableMap) value;
  float red = (float) map.getDouble("red");
  float green = (float) map.getDouble("green");
  float blue = (float) map.getDouble("blue");
  float alpha = (float) map.getDouble("alpha");
  int colorSpace = map.getInt("colorSpace")

  if (colorSpace === 1) {
@ColorLong long p3 = pack(red, green, blue, alpha, ColorSpace.Named.DISPLAY_P3);
    return Color.valueOf(p3);
  } else {
    // ...
  }
}
```

### Docs

Update the [Color Reference](https://github.com/facebook/react-native-website/blob/main/docs/colors.md) to document color function usage in React Native.

## Drawbacks

There should be no breaking changes for users but the color implementation will be a bit more complicated than before.

## Alternatives

We could do nothing and require users to patch package for Display P3 color support. But this is a disappointing developer experience and not a satisfying solution.

## Adoption strategy

This is intended to be a non-breaking change with zero impact to existing users. Adoption is entirely opt-in by using the color function in styles to start using display-p3 colors.

## How we teach this

Developers familiar with [color() on web](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color) should find this feature familiar and straightforward to use. To ensure that this feature is easily accessible to all developers we should do the following:

1. Document the inclusion of the color function in the [official color reference](https://reactnative.dev/docs/colors).
1. Partner with the Release Crew to make sure the new feature is included in a release blog post.
1. Publish a blog post on the [Infinite Red blog](https://shift.infinite.red/)

## Unresolved questions

- While Android does support wide color gamut, do [Android View](<https://developer.android.com/reference/android/view/View#setBackgroundColor(int)>)'s actually support it? These methods take `int`s while the Android color reference states wide gamut colors are `long`s.
- Do we need to do additional work to support interpolating colors for animations?
