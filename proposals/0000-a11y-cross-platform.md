---
title: Cross-platform Accessibility Improvements
author:
- Marc Mulcahy
date: 2018-10-31
---

# RFC0000: Cross-platform Accessibility Improvements

## Summary

Several accessibility-related properties are only available on one platform (either iOS or Android, but not both). This proposal attempts to make as many properties available on both Android and iOS as possible.

Properties only available on iOS include:

* accessibilityViewIsModal
* accessibilityElementsHidden

Properties only available on Android include:

* accessibilityLiveRegion

Several properties are missing from both platforms:

* accessibilityCompoundComponent: replace accessible-- indicates to assistive technologies that all the components inside this one should be treated as a single component.
* accessibilityRelationships: describes how this component is related to others
* accessibilityRange: expose information about range-based components such as sliders

## Motivation

All accessibility properties should be available on all supported platforms. This reduces the burden on developers to understand the nuances of accessibility support on various platforms. It also allows JavaScript components to expose rich accessibility information, rather than relying on the accessibility support in native components.

## Detailed design

### accessibilityViewIsModal

#### Android/Fire OS

When a component marks itself as modal, code in the view manager will need to traverse all other components in the view tree and mark them as importantForAccessibility=no. A window content changed event with content change types set to subtree will then need to be fired on the root window to ensure that assistive technologies receive the tree updates.

It may also be necessary to install a view tree observer once the modal component is active to prevent background components from appearing. We'll need to do some real-world testing to understand how realistic this scenario might be, and whether it's worth the performance cost of managing such an observer.

#### iOS

This property is currently supported on iOS, so no change to iOS support is needed.

### accessibilityElementsHidden

#### Android/Fire OS

Supporting this property in Android may be as simple as marking the component as importantForAccessibility=noHideDescendants

If this doesn't match the iOS behavior, we may need to build an accessibility delegate to hide the children. This will likely be tricky, since the children will still be generating events.

#### iOS

This property is already supported on iOS, so no iOS changes are required.

### accessibilityLiveRegion

This allows components to tell assistive technologies how to react to changes in their content.

#### Android/Fire OS

This property is already supported on Android, so no changes in Android support are required.

#### iOS

We believe this functionality can be implemented using accessibility notifications on iOS. For assertive live regions, the updated text of the component can be sent immediately to iOS as an announcement notification. For polite live regions, code can wait for the AnnouncementDidFinish notification before posting the updated text as an announcement notification.

### accessibilityCompoundComponent

We think the current implementation, which is called "accessible", should be renamed to accessibilityCompoundComponent, to more accurately represent what it does. The functionality should remain the same.

### accessibilityRelationships

This property describes how this component is related to others. Relationship types include:

* labelFor: This component labels the specified component
* labeledBy: this component is labeled by the specified component
* describedBy: this component is described by the specified components.

When referring to other components, we may need to add an additional property to view-- namely accessibilityId. We need to find the best forward-looking approach for referencing other components.

#### Android/Fire OS

For labelFor and labeledBy, use the standard Android view and AccessibilityNodeInfo methods.

For describedBy, which is only supported on Fire OS, put the appropriate key/values in the extras bundle.

#### iOS

When associating labels with components, include the label text in the target component's accessibilityLabel.

Don't support describedBy on iOS for now. In future, we could explore adding the accessibilityLabels of the describedBy targets to the accessibilityHint of the component.

### accessibilityRange

This property allows components (such as sliders) to expose range-based information. The value of the property is an object containing the following values:

* min: the minimum value of the range
* max: the maximum value of the range
* current: the current value of the component (within the range specified by min and max)

#### Android/Fire OS

This maps directly to Android's RangeInfo interface.

#### iOS

iOS supports a property called accessibilityValue. This value is a simple string. accessibilityRange will be converted to a localized string and placed into accessibilityValue.

## Drawbacks

It will be challenging to ensure that the behavior of these properties is the same between iOS and Android. Building cross-platform properties will likely involve sacrificing a small amount of functionality on each target platform to ensure consistency.

## Alternatives

Components could continue to use the platform-specific properties. This has the drawback of requiring developers to have at least a cursory
understanding of accessibility on both iOS and Android. And in the case of accessibilityViewIsModal, we expect this to be somewhat difficult to implement correctly on Android.

## Adoption strategy

We don't believe this to be a breaking change. Developers who currently use these properties on Android or iOS can continue to do so, and remove conditional use to allow functionality on other platforms.

## How we teach this

The same documentation that is used to describe the properties on either iOS or Android can be marked as cross-platform for use on either. For the accessibilityRelationships property, labeledBy and labelFor are common paradigms across accessibility support on multiple platforms including web.

## Unresolved questions

* Exactly what is the behavior for accessibilityElementsHidden on iOS? Does this hide the component itself from accessibility, or only its children?
* What is the best mechanism for one component to refer to another for the purpose of specifying labels, targets, descriptive components, etc.?
