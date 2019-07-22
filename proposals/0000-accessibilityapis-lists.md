---
title: Accessibility States and Roles for lists
author:
- Harini Kannan
date: 06/12/2019
---

# RFC0000: Accessibility States and Roles for lists

## Summary

This proposal adds new accessibility roles for supporting lists as well as new APIs for describing size of lists,nth item in the list etc.,

This addresses Issues [2578](https://github.com/microsoft/react-native-windows/issues/2578) and [2579](https://github.com/microsoft/react-native-windows/issues/2579) from react-native-windows. An implementation of the same can be tracked through [this PR](https://github.com/microsoft/react-native-windows/pull/2743). 

## Basic example

React Native developers can use these new APIs to describe lists, listitems, size of lists, nth item in the list etc., to assistive technologies using simple code as shown below:

```
<FlatList
  data={this.state.listdata}
  accessibilityRole="list"
  renderItem={({item}) =>
     <View accessibilityRole="listItem" 
           accessibilitySetSize ={this.state.listdata.count}
           accessibilityPostInSet={item.id} >
        <Text>{item.name}</Text>
     </View>
  }
</FlatList>
```


## Motivation

It is currently not possible for react-native developers to describe size of lists, nth item in the list etc., With these new APIs, react-native developers can provide more information on lists to assistive technologies which can then be more descriptive of list experiences. 

## Detailed design

### P1 APIs:
The following roles will be added to [accessibilityRole](https://facebook.github.io/react-native/docs/accessibility#accessibilityrole-ios-android) APIs:
- **list** Used when element should be treated as a list
- **listitem** Used when element should be treated as a list item

FlatList, SectionList, VirtualizedList components will have `accessibilityRole = list` set by default. 

### P2 APIs:
The following new properties will be introduced to customize and describe details of the list/list items:
- **accessibilitysetsize** 
- **accessibilityposinset**

In keeping with the ARIA equivalents like [aria-setSize](https://accessibilityresources.org/aria-setsize) and [aria-posInSet](https://accessibilityresources.org/aria-posinset), these properties are marked on the members of a set, not the container element that collects the members of the set. When set, assistive technologies will call out "item X out of Y," when a member of a set with this property has accessibility focus. The assistive technologies would use X equal to the `acessibilityPosInSet` attribute and Y equal to the `accessibilitySetSize` attribute.

These properties should be used in conjunction with each other.

## Drawbacks

FlatList tries to be a simple building block component and hence adding a lot of functionality to it can make it more complex. However, the APIs we are adding are ARIA based and have equivalents in all platforms. However, Lists are also some of the most common experiences in any app and hence the small amount of complexity this adds feels warranted.

Given the nature of FlatList, SectionList in React Native where there are no actual collections/items that the list itself maybe aware of, it is not possible to provide good default behaviors for the item specific APIs in this proposal and it is up to each developer to set these values for the right behaviors.

## Adoption strategy

These APIs all have existing equivalents in ARIA, UIA and other native accessibility APIs. The naming of these has been matched to existing ARIA equivalents and hence will be familiar to React/Web developers.

For Windows developers, these should be familiar as well - the UIA properties that describe these for accessibility tools are [PositionInSet](https://docs.microsoft.com/en-us/uwp/api/Windows.UI.Xaml.Automation.AutomationProperties.PositionInSetProperty) and [SizeOfSet](https://docs.microsoft.com/en-us/uwp/api/Windows.UI.Xaml.Automation.AutomationProperties.SizeOfSetProperty)

## How we teach this

We should add these to the existing [Accessibility documentation](https://facebook.github.io/react-native/docs/accessibility) in react-native. 

## Unresolved questions

Can these values be set on any component? 
