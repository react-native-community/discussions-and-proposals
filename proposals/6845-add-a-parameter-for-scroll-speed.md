---
title: Add a parameter for scroll speed
author:
- Andrey Fedorovich
date: 30.01.2024
---

# RFC6845: Add a parameter for scroll speed

## Summary

To manipulate the scroll animation speed, developers now need to create the animation themselves instead of using it out of the box

## Basic example

The possible way to implement that functionality could be:

```javascript
scrollToIndex: (params: {
  index: number;
  animated?: boolean;
  viewOffset?: number;
  viewPosition?: number;
  speed?: number; // new value 
});
```

## Motivation

Usage of FlatList scrollToIndex function with an animation speed parameter improves the developers' experience

## Detailed design

### 1. Components: FlatList, VirtualizedList
#### Update Methods
- scrollToIndex
- scrollToEnd
- scrollToItem
- scrollToOffset

### 2. Component: SectionList
#### Update Methods
- scrollToLocation

### 3. Component: ScrollView
#### Update Methods
- scrollTo
- scrollToEnd

## Alternatives

An alternative to implementing a new parameter is to create a scroll move mechanism manually. This would involve duplicating the logic of the scrollTo function. The first implementation would be handled in React Native core, while the second one would be implemented by developers. They would likely need to know the scroll wrapper's height or width, as well as the scroll item's height or width, to move their users to the desired position

## Adoption strategy

To use the new property in scrollable elements, users need to update their React Native version and extend existing functions with the `speed` parameter

## How we teach this

To explain users new opportunity, in the documentation should be reminder:

⚠️ The `speed` parameter is available starting from React Native version x.y.z.
