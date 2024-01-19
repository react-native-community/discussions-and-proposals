---
title: Hover style API for iPad / Apple Vision Pro
author:
  - Oskar Kwaśniewski
  - Saad Najmi
date: 08.01.2023
---

# RFC0000: Hover style API for iPad / Apple Vision Pro (Cursor: pointer)

## Summary

iPad supports hover style effects when using an external mouse/trackpad. This effect illuminates the UIView in two configurable ways: lift or highlight. This proposal intends to add support for this API in React Native. This API is also used by visionOS to display the hover effect when a user looks at the view.

The native API this feature is basing on: https://developer.apple.com/documentation/uikit/uihoverstyle (iOS 17+). There is an older version of this API supporting iOS 13.4+ https://developer.apple.com/documentation/uikit/uipointerstyle

## Basic example

This effect would be applied by adding `cursor: pointer` style to a view (which is used to render touchable/pressable elements).

```jsx
<Pressable style={{ cursor: "pointer" }} onPress={onPressFunction}>
  <Text>I'm pressable!</Text>
</Pressable>
```

The hover effect automatically adapts to view’s corner radius.

Example of highlight effect:

https://github.com/react-native-community/discussions-and-proposals/assets/52801365/a1ace4ac-44c1-45e6-afbe-6e0b0f05f74d

## Motivation

Controlling hover style allows React Native developers to provide a much better user experience on iPad and Apple Vision Pro, allowing them to seamlessly interact with React Native applications using the mouse.

Additionally, We have lots of React Native experiences in brownfield contexts at Microsoft. The value add of React Native over something like a webview is that we can access the system APIs, and provide the best UX for the platform. This includes iPadOS / visionOS, where hover events are now a part of the UX language and built into UIKit controls. In order for React Native experiences to thrive living side by side with native experiences, we would live to have as close UX as possible.

This API can work side by side with the Pointer Events API. Users can choose to either spin up their own custom interactions with the pointer events API, or use Apple OS default. Adding a hover effect shouldn’t affect pointer events being fired or not. https://reactnative.dev/blog/2022/12/13/pointer-events-in-react-native

Also in react-native-macos the `cursor: pointer` API is already present so this will introduce a familiar API to core.

## Detailed design

_Disclaimer: I will discuss the design for New Architecture, but the API can be easily backported._

Also this API is already working in react-native-visionos [fork](https://github.com/callstack/react-native-visionos/blob/58f6e180d49cbd638ff9b637c0308808eeee2321/packages/react-native/React/Fabric/Mounting/ComponentViews/View/RCTViewComponentView.mm#L522).

The first version of this api should support adding only `UIHoverAutomaticEffect`, which later on can be expanded and other hover effects can be exposed as `-rn-apple-lift` and `-rn-apple-highlight`.

The API should be opt-in to avoid breaking current iPad apps. The implementation bases on extending `RCTView` with a new prop of type `std::string`.

1. Add `cursor` property to RCTView And list it in `Libraries/StyleSheet/StyleSheetTypes.js`

2. Handle setting UIView `hoverStyle` property with a simillar function to the one below.

```objc
- (void) updateHoverEffect:(NSString*)hoverEffect {
    if (hoverEffect == nil || [hoverEffect isEqualToString:@""]) {
        self.hoverStyle = nil;
        return;
    }

    UIShape *shape = [UIShape rectShapeWithCornerRadius:self.layer.cornerRadius];
    id<UIHoverEffect> effect;

    if ([hoverEffect isEqualToString:@"lift"]) {
        effect = [UIHoverLiftEffect effect];
    } else if ([hoverEffect isEqualToString:@"highlight"]) {
        effect = [UIHoverHighlightEffect effect];
    }

    if (hoverEffect != nil) {
        self.hoverStyle = [UIHoverStyle styleWithEffect:effect shape:shape];
    }
}
```

Here we initialize a new `UIShape` which reassembles the current’s view layer `cornerRadius`. Next we are creating a `UIHoverEffect` instance based on the value passed by user.

The current API implementation might need some polish to correctly retrieve corner radius.

## Drawbacks

- Apple only API
  - Only Android equivalent I found is: https://developer.android.com/reference/android/view/MotionEvent which doesn’t provide any built-in effect but allows to create custom ones.
- There might be different plan for the Pointer Events API.

## Alternatives

- This feature could be also implemented in user space by creating their own subclass of UIView and wrapping every element with it.

## Adoption strategy

Users can opt-in without any breaking changes.

## How we teach this

Describe this prop in the documentation. Add section to docs about iPad development.

## Unresolved questions
