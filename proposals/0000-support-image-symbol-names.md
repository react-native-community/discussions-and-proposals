---
title: Support Image Symbol Names
author:
- Aaron Brager
date: 2024-02-11
---

# RFC0000: Support Image Symbol Names

## Summary

This proposal suggests adding support for custom symbol images in React Native's `Image` component. Currently, React Native supports loading images from local files and network URLs. However, there's no direct support for using custom symbol images on iOS/macOS/tvOS. This feature would improve developers' ability to use React Native to create a native look and feel without relying on third-party libraries.

If you're not familiar with this iOS feature there's more info here: https://developer.apple.com/documentation/uikit/uiimage/configuring_and_displaying_symbol_images_in_your_ui

## Basic example

There's a few different approaches for the API and I'm not married to this one, but it'd look something like this:

```javascript
import React from 'react';
import { View, Image } from 'react-native';

const CustomSymbolImageExample = () => (
  <View>
    <Image source={{ systemSymbol: { name: 'custom.multiply.circle', configuration: {
      weight: 'thin',
      scale: 'large'
    }}} />
  </View>
);

export default CustomSymbolImageExample;
```

## Motivation

Currently, developers resort to workarounds like using third-party libraries such as `react-native-sfsymbols` to do what should be built-in. This change would reduce dependencies on external packages and provide a more cohesive ecosystem.

## Detailed design

Extend React Native's `Image` component to accept system symbol names. This component would bridge to e.g. `UIImage(systemName: "multiply.circle.fill")` to load the correct symbol, then render it in a `UIImageView`, which is similar to how the current implementation works, except with a different initializer.

Android doesn't support an equivalent.

## Drawbacks

- Implementation cost: Adding support for custom symbol images may introduce additional complexity to the `Image` component implementation.
- Platform specificity: This would only be available on Apple platforms which would exacerbate platform disparities in React Native apps.

## Alternatives

- Use `react-native-sfsymbols` 
- Handwrite a bridge

## Adoption strategy

It's not a breaking change. We could optionally coordinate with `react-native-sfsymbols` to use the new functionality under the hood in their library. We could also invite them to contribute their source code to the core React Native project.

## How we teach this

Nothing major would change about teaching this, just some additions to the docs.

## Unresolved questions

- Should we support a bunch of separate props to compose the system symbol names, have typescript enums, etc? (In my example above I just used strings.)
- Some symbols support animations, what should the API interface be for that? (I think we could still use `Image` for this.)
