---
title: Improvements to accessibilityRole and accessibilityStates
author:
- Marc Mulcahy
date: 2019-02-06
---

# RFC0054: Improvements to `accessibilityRole` and `accessibilityStates`

## Summary

Some brilliant work was begun to add cross platform support for accessibility roles and states to React Native. We propose to add some additional roles and states, as well as extend the implementation on Android and Fire OS.

## Basic example

With the current set of roles, it is not possible to describe a checkbox. These changes would enable this scenario, among other things:

    <View
      accessibilityRole="checkbox"
      accessibilityStates={['unchecked']}>
      <Text>Show Password</Text>
    </View>

## Motivation

When developers build components in JavaScript, they need the ability to express the role and current state(s) of the component to assistive technologies running on the platform.

## Detailed design

### New Roles

This proposal adds the following roles:

- alert
- checkbox
- combobox
- editabletext
- menu
- menubar
- menuitem
- progressbar
- radiobutton
- radiogroup
- scrollbar
- spinbutton
- switch
- tab
- tablist
- timer
- toolbar

### New States

This proposal adds the following states:

- on
- off
- checked
- unchecked
- busy
- expanded
- collapsed
- hasPopup

### Fallback Mechanism for Roles

There are several roles which don't have native analogs on a given platform. For such roles, a fallback mechanism will be used.

#### Android

The fallback mechanism is already in place on Android. For roles which don't have standard Android view analogs, the class name of the AccessibilityNodeInfo is set to the class name of the closest native Android view type, and the roleDescription extra is set to a localized textual description of the role.

### Fire OS

For Fire OS, we will always embed the role as a key/value pair in the AccessibilityNodeInfo's extra bundle.

key: com.amazon.accessibility.role
value: the role string

The role string should not be localized-- it should be one of the constants defined above, or in the existing documentation. This will allow assistive technologies on Fire OS to consume the role directly.

#### iOS

For roles which don't have a corresponding iOS accessibilityTrait, we recommend appending a localized version of the role description to the accessibilityValue of the component.

### Additional States Implementation Notes

Android has slightly more robust state support than does iOS. The following states can be represented with Android'd standard accessibility API:

- checked
- unchecked
- expanded
- collapsed
- hasPopup

We will append a localized description of other states to the component's content description.

on iOS, a localized description of states which do not have corresponding accessibilityTraits will be appended to the component's accessibilityValue.

## Drawbacks

Supporting these additional roles and states will necessarily make the platform accessibility implementations a bit more complex, especially when supporting roles and states which have no native analog on a given platform.

We do not see this impacting any existing code. Developers who choose to use the new roles and states will get improved accessibility functionality. Those who choose not to will see no change in the behavior of their existing code.

## Alternatives

The only alternative is to recommend that developers append the role and state information to the text of the accessibility label. This does not allow a disabled user to control how much, and in what order, they hear information about a component. It also does not support compact rendering of role and state information in braille. It also requires localization effort in each component which employs it.

## Adoption strategy

We should document the additional roles and states as part of the official accessibility API documentation. Developers can then start using them if they choose.

## How we teach this

The attempt with this proposal is to bring React Native accessibility support one step closer to generally accepted accessibility idioms and practices as specified by standards such as ARIA. As such, we believe this will make it easier for developers familiar with accessibility on the web or other platforms to relate React Native's accessibility support to what they already know.

## Unresolved questions

- Roles imply supported states and behaviors. For example, checkboxes imply the checked state, and the ability to be toggled. Radio groups typically only allow one child to be selected, etc. When a component specifies its role, should this programmatically require the developer to implement certain behaviors? Alternatively, what mechanisms exist to present warnings during compile time for missing behaviors?
