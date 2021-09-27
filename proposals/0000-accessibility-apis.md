---
title: Accessibility APIs
author:
- Nicolas Gallagher
date: 2021-09-20
---

# RFC: Accessibility APIs

## Summary

This is a proposal to expand React Native accessibility APIs, and to align those APIs with equivalent web standards, e.g., [WAI-ARIA](https://www.w3.org/TR/wai-aria-1.2/). The aim is to provide developers with a single, familiar accessibility model rather than requiring deep knowledge of the models of each target platform.

## Motivation

To provide React Native developers with a single accessibility model that allows for targeting a wide range of platforms, rather than requiring deep knowledge of the models of each target platform.
To continue betting on React Native's web origins, as most React Native users have a web background and many of the existing APIs are familiar to web engineers. To avoid designing competing UI standards, and instead leverage existing specs as far as possible, e.g., WAI-ARIA.

This proposal also meets the motivations for previous accessibility proposals related to [lists](https://github.com/react-native-community/discussions-and-proposals/blob/ef368d499b9350c8bcfbfdc8db23b8592982f123/proposals/0000-accessibilityapis-lists.md) and [relationships](https://github.com/react-native-community/discussions-and-proposals/blob/7f6fe5fdba15e4ecd147b69110e81aaf0719e5c4/proposals/0000-a11y-cross-platform.md).

## Detailed design

The web has well-defined accessibility requirements to meet the needs of people using the broad range internet enabled devices and input hardware. Supporting these standards should in turn [benefit native](https://reactnative.dev/blog/2021/08/26/many-platform-vision#embracing-platform-constraints) app accessibility. The accessibility paradigms across Android, iOS, Web, Windows, and macOS have similarities but are not entirely equivalent. A significant challenge lies in shielding users of React Native's accessibility APIs from the expertise otherwise needed to target multiple platforms at once. The rest of this section focuses on specific accessibility concerns and outlines proposed APIs.

### Accessible focus

React Native needs a clear mechanism for marking up elements as focusable by a user action or assistive technology. The `focusable` prop in React Native is currently an Android-only prop to markup elements as *keyboard* focusable. React Native for Web and React Ntive for Desktop use `focusable` to markup elements as *user* focusable, whether by keyboard focus (i.e., appearing in the tab-flow) or as a result of pointer interaction. Elements marked as `focusable={false}` should also provide semantic information, e.g., hiding natively focusable elements from the keyboard tab-flow.

### Accessible lists

React Native needs a mechanism for marking up elements as lists and list items, along with the ability to describe the size of lists and the position of a list item in the set. React Native for Web supports `accessibilitySetSize` and `accessibilityPosInSet` for this purpose.

### Accessible state

React Native currently includes the [`accessibilityState`](https://reactnative.dev/docs/accessibility#accessibilitystate) and [`accessibilityValue`](https://reactnative.dev/docs/accessibility#accessibilityvalue) props to work with limited states and value ranges respectively. However, both of these props are objects. This makes it harder to implement hooks that manage specific accessibility state or value props, as it often requires careful merging of fields that may be managed by different functions. Using objects to manage a mix of static and dynamic properties also has worse perfomance characteristics as updating the value of `accessibilityValue.now` means recreating the entire `accessibilityValue` object, even though the other properties are static. Additional states (e.g., pressed) are not represented in the React Native `accessibilityState` prop. React Native for Web supports all the state and value fields as top-level, primitive-type props, e.g., `accessibilityValueNow`.

### Accessible relationships

React Native would benefit from a way to create accessibility relationships between elements, linked by `nativeID`. React Native for Web has included [`accessibilityLabelledBy`](https://www.w3.org/TR/wai-aria-1.2/#aria-labelledby) and [`accessibilityDescribedBy`](https://www.w3.org/TR/wai-aria-1.2/#aria-describedby) props, which accept an ID or an array of IDs. These props are used in conjunction with `nativeID` to define accessibility relationships between (multiple) elements.

### Phase 1

Renaming existing APIs.

**Example diffs**. Renaming `autoCompleteType` to `autoComplete`: [8066bc9ff](https://github.com/facebook/react-native/commit/8066bc9ff6a3f1d6910ccd346ce0fe6b7b4b16fc), [27fec9569](https://github.com/facebook/react-native/commit/27fec9569e08a04e0dbdbd5de063a599ad0416fa).

* Add `accessibilityBusy: boolean`. Replaces `accessibilityState.busy`.
    * Add `accessibilityBusy` prop with docs. Spec. 
    * Deprecate `accessibilityState.busy`.
* Add `accessibilityChecked: boolean | 'mixed'`. Replaces `accessibilityState.checked`.
    * Add `accessibilityChecked` prop with docs. Spec.
    * Deprecate `accessibilityState.checked`.
* Add `accessibilityDisabled: boolean`. Replaces `accessibilityState.disabled`.
    * Add `accessibilityDisabled` prop with docs. Spec. 
    * Deprecate `accessibilityState.disabled`.
* Add `accessibilityExpanded: boolean`. Replaces `accessibilityState.expanded`.
    * Add `accessibilityExpanded` prop with docs. Spec. 
    * Deprecate `accessibilityState.expanded`.
* Add `accessibilitySelected: boolean`. Replaces `accessibilityState.selected`.
    * Add `accessibilitySelected` prop with docs. Spec. 
    * Deprecate `accessibilityState.selected`.
* Add `accessibilityValueMin: number`. Replaces `accessibilityValue.min`.
    * Add `accessibilityValueMin` prop with docs. Spec. 
    * Deprecate `accessibilityValue.min`.
* Add `accessibilityValueMax: number`. Replaces `accessibilityValue.max`.
    * Add `accessibilityValueMax` prop with docs. Spec. 
    * Deprecate `accessibilityValue.max`.
* Add `accessibilityValueNow: number`. Replaces `accessibilityValue.now`.
    * Add `accessibilityValueNow` prop with docs. Spec. 
    * Deprecate `accessibilityValue.now`.
* Add `accessibilityValueText: string`. Replaces `accessibilityValue.text`.
    * Add `accessibilityValueText` prop with docs. Spec. 
    * Deprecate `accessibilityValue.text`.
* Add `accessibilityHidden: boolean`. Replaces `accessibilityElementsHidden` (iOS) / `importantForAccessibility` (Android).
    * Add `accessibilityHidden` prop with docs. Spec. 
    * Deprecate `accessibilityElementsHidden` and `importantForAccessibility`.
* Add `accessibilityLiveRegion` value of `"off"`. Replaces `"none"`.
    * Add `"off"` value.
    * Deprecate `"none"` value.
* Add `accessibilityModal: boolean`.  Replaces `accessibilityViewIsModal` (iOS).
    * Add `accessibilityModal` prop with docs. Spec. 
    * Deprecate `accessibilityViewIsModal`.
    * Add Android support.
* Add `accessibilityRole` value of `"slider"`. Replaces `"adjustable"`.
    * Add `"slider"` value.
    * Deprecate `"adjustable"` value.
* Add `accessibilityRole` value of `"heading"`. Replaces `"header"`.
    * Add `"heading"` value.
    * Deprecate `"header"` value.
* Add `accessibilityRole` value of `"img"`. Replaces `"image"`.
    * Add `"img"` value.
    * Deprecate `"image"` value.
* Remove `accessibilityHint` and recommend using `accessibilityLabel` instead.
    * Deprecate `accessibilityHint` and update docs.
* *Codemods*
    * Codemod for accessibility prop and value changes.

### Phase 2

Adding and repurposing APIs.

* Add `accessibilityActiveDescendant: IdRef` prop.
  * Linked to `nativeId`.
* Add `accessibilityDescribedBy: IdRefList` prop.
  * Reflist. Accept array of ids. Linked to `nativeId`.
* Add `accessibilityHasPopup: boolean` prop.
* Add `accessibilityLabelledBy: IdRefList` prop.
  * Reflist. Accept array of ids. Linked to `nativeId`.
* Add `accessibilityLevel: number` prop.
    * iOS equivalent is [`accessibilityTextHeadingLevel`](https://developer.apple.com/documentation/foundation/nsattributedstring/key/2865779-accessibilitytextheadinglevel). Note: on web `accessibilityLevel` can be used with roles other than heading, so iOS should have a dependency on the correct role.
* Add `accessibilityPosInSet: number` prop.
* Add `accessibilityPressed: boolean` prop.
* Add `accessibilityRole` values.
    * Expand the set of allowed values.
    * Values to add: alertdialog, application, article, banner, blockquote, caption, cell, code, columnheader, combobox, complementary, contentinfo, definition, deletion, dialog, document, emphasis, feed, figure, form, generic, grid, gridcell, group, insertion, list, listbox, listitem, log, main, math, meter, menuitemcheckbox, menuitemradio, navigation, note, option, paragraph, range, region, row, rowgroup, rowheader, searchbox, separator, status, strong, subscript, superscript, table, tabpanel, term, textbox, time, tree, treegrid, treeitem.
    * Implement Android/iOS support for values with native equivalents.
* Add `accessibilitySetSize: number` prop.
* Redefine `focusable` semantics.
    * The focusable prop should markup elements that can hold the input focus, whether focus is triggered via keyboard, pointer, or screen-reader. Behavior is as equivalent to `tabIndex=0|-1`. This is an abstraction over W3C the [`tabIndex`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex) attribute that enforces a boolean API to prevent the accessibility anti-pattern of defining explicit tab orders.
    * Determine whether `focusable={true}` should warn if used with a non-interactive accessibilityRole, as assistive technologies may not otherwise surface the element in the accessibility tree.

There are additional ARIA attributes that could be included as props, e.g., `aria-rowspan`, etc. However, we don't have an immediate need for them and can revisit whether they should be integrated at a later date.

### Phase 3

Removing platform-specific APIs.

* Remove `accessible` (iOS).
* Remove `accessibilityState`.
* Remove `accessibilityValue`.
* Remove `accessibilityElementsHidden` (iOS) and `importantForAccessibility` (Android).
* Remove `accessibilityViewIsModal` (iOS).
* Remove `accessibilityLiveRegion` value of `"none"`.
* Remove `accessibilityHint` (iOS).
* Remove `accessibilityRole` values of `"adjustable"`, `"header"`, `"image"`.
* Remove `accessibilityIgnoresInvertColors` (iOS).

## Drawbacks

The cost to renaming existing props is relatively low. Adding new features is dependent on feedback from Android/iOS accessibility expertise and the viability of implementing the ARIA model on these platforms. Codemods would be useful to help update the community's use of deprecated props.

## Alternatives

Designs that are substantially different include: requiring developers to provide different accessibility props for each platform; and designing a proprietary cross-platform accessibility API. The approach detailed above was chosen as it avoids the need to for us to develop a proprietary API, better leverages the existing expertise of developers familiar with the WAI-ARIA open standard, and allows for an efficient web implementation.

## Adoption strategy

The proposed accessibility APIs can be introduced incrementally and codemods can be used to automate some of the prop transforms. Existing accessibility props can be deprecated once their replacements have been introduced and validated. Developers can reference the WAI-ARIA spec both to implement and use these APIs.

## How we teach this

The React Native accessibility docs would need to be altered. Existing resources related to WAI-ARIA can be applied to accessibility in React Native.

## Unresolved questions

How to incorportate Android/iOS "accessibility events"? How to implement a web-centric accessibility API within the iOS accessibility paradigm?
