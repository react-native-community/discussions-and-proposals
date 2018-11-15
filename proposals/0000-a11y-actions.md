---title: Accessibility Actions Support
author:
- Marc Mulcahy
date: 2018-10-31
---

# RFC0000: Accessibility Actions Support

## Summary

Custom components often require interactions beyond simple touches. Programatic access to these interactions must be provided to allow disabled users to perform them.

## Basic example

Consider the simple example of a custom slider. The newly expanded accessibilityRole property allows us to indicate that the component is a slider in a cross-platform way. However, the blind or motor impaired user can't adjust the slider, since a gesture other than a single tap is required to do so.

## Motivation

The example above is one use case, but there are several common patterns and use cases which require custom action support. Here are a few examples:

* Adjustable controls (such as sliders)
* Scrollable containers where the scrolling is not handled natively
* Extra swipe actions in a list (swipe right to delete an email message for example)

## Detailed design

This proposal adds an additional accessibility-related property called accessibilityActions to View. The value of this property is a list of accessibility action objects. Each action contains the following properties:

* name: a string name for the action. Note that a pre-defined set of standard actions is supported, but components can also define their own custom actions.
* description: a localized description of the action which can be presented to the user to describe what will happen when the action is performed.
* function: the function to call when the assistive technology requests this action to be performed. This function should return true if the action is performed, and false if not.

  Standard action names include:

* activate: activate the item (typically accomplished by double tapping the item when using a screen reader)
* longpress: Long presses anywhere inside the component
* scrollforward: Scrolls the component forward one screen full
* scrollbackward: Scrolls the component backward one screen full
* dismiss: Dismisses the component
* magictap: on iOS, user double taps with two fingers while accessibility focus is on the component. Not available on Android.
* adjustup: Adjusts component up
* adjustdown: adjust component down
* expand: Expands the component
* collapse: Collapses the component
* copy: copies content in the component to the clipboard
* cut: cuts content in the component to the clipboard
* paste: pastes clipboard contents into this component
* setvalue: sets the value of this component (typically used to adjust sliders, etc.)

### Implementation Notes

#### Android/Fire OS

A custom AccessibilityDelegate will handle custom actions which are added to React Native components. This concept maps fairly directly to Android's notion of AccessibilityActions.

#### iOS

React Native actions will be implemented on iOS via the UIAccessibilityAction informal protocol. Standard actions will be used to support React Native standard actions where appropriate. iOS custom actions will be used to support React Native custom accessibility actions.

## Drawbacks

* Difficult for new developers to know what actions they should support.
* Localization of action descriptions seems problematic and hard to maintain.
* Any performance concerns making the action functions synchronous?

## Alternatives

For native components, the platform often provides the necessary actions or other support to allow assistive technologies to manipulate the components. This is only required for custom components built entirely in JavaScript. But for such components, I don't know of another solution.

## Adoption strategy

This is not a breaking change. Components can migrate to the new API as needed.

## How we teach this

The simple rule of thumb is that if your component can be manipulated in ways other than a simple touch, it needs custom action support. The tricky bit to teach will be exactly what actions should you support, especially for scrolling containers.

## Unresolved questions

* What is the most efficient way to localize the description strings? Should they be IDs rather than actual strings?
* How will we add custom actions to native components?
* What additional standard actions should be supported?
* When an action supports arguments, how should we communicate the meaning of those arguments to assistive technologies? Do we rely on the assistive technologies knowing about particular actions and their arguments out of band?
* Should we namespace actions to avoid collisions? For example, if you're defining your own copy action, should the name be "com.example.copy"?
