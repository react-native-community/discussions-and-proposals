---
title: Add support for CSS color() function
author:
  - Ryan Linton
  - Yulian Glukhenko
date: 2023-11-20
---

# RFC0000: Add support for CSS color() function

## Summary

React Native does [not currently support](https://github.com/facebook/react-native/issues/41517) wide color gamut color spaces (e.g. display-p3). This proposal discusses adding support for the CSS color() function enabling support for additional wide-gamut color spaces.

## Basic example

```js
StyleSheet.create({
  backgroundColor: "color(display-p3 1 0.5 0)",
  color: "color(display-p3 1 0.5 0 / .5)",
});
```

## Motivation

Since most new devices support wide-gamut color spaces React Native should support them as well. The Display P3 color space has had native support since Android 8.0 and iOS 9.3. The color() function was introduced with [CSS Color Module Level 4](https://drafts.csswg.org/css-color/#color-function), much of which is already implemented in React Native. Also Flutter [recently added support](https://github.com/flutter/flutter/issues/55092) for Display P3 on iOS with plans to follow up with support for Android and then the framework itself.

## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

## Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React Native
- integration of this feature with other existing and planned features
- cost of migrating existing React Native applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? Why did you select your approach?

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

## How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers?

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?
