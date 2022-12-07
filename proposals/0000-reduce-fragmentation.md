---
title: Reduce API fragmentation across platforms
author:
- Nicolas Gallagher
date: 2022-08-03
updated: 2022-11-29
---

# RFC: Reduce API fragmentation across platforms

## Summary

This is a proposal to incrementally reduce the API fragmentation faced by developers using React to target multiple platforms with code shared between native and web. The proposed cross-platform user interface APIs are a subset of existing web standards for DOM, CSS, and HTML - "Strict DOM". Each proposed change is additive and does not require migration of existing React Native UI code (deprecations are optional / follow up work). Incremental progress will reduce the fragmentation between React Native and React DOM components.

## Motivation

React Native currently includes many APIs that are modelled on Web APIs but do not conform to the standards of those Web APIs. React Native also includes many APIs that achieve the same results on Android and iOS but are exposed as 2 different props. And React Native includes several APIs that have known performance (network and runtime) drawbacks on Web.

This proposal aims to allow developers to target more platforms with cross-platform APIs, and deliver better performance when targeting browsers. Features for Android, iOS, and Web are unified by aligning with the Web standard. Support standards helps to:

* minimize the overhead when running in browsers;
* reduce developer education required to learn features;
* set clear and cohesive API end-state expectations for contributors to aim for;
* support backwards compatibility;
* develop universal codebases.

React Native for Web uses React DOM and native DOM APIs to shim React Native components and APIs on the web. The tooling for an existing React DOM app simply maps the 'react-native' export to 'react-native-web'. This is the most complete and widely used shim, but comes with considerable DX and UX costs on the web.

![](https://user-images.githubusercontent.com/239676/205388174-6fce19f1-8b3a-4c6e-82f1-89ef9c21ab7d.png)

Implementing a Strict DOM subset in React Native shifts the weight of bridging native and web apps onto React Native Native, where it can be done most efficiently. On the web, we would combine React DOM with stylex.

![](https://user-images.githubusercontent.com/239676/205388313-e58f8793-51cd-4a6e-86e7-89425fd2d0e1.png)

## Detailed design

The overall work involved is broken down into 5 sections.

* [Props API](#props-api-react-dom-subset). React DOM component props to add to existing React Native components.
* [Styling API](#styles-api-css-subset). CSS APIs to add to React Native environment.
* [Environment API](#environment-api). Global APIs to add to the host environment.
* [Components API](#components-api-dom-subset). React DOM components to add to React Native.
* [Elements API](#elements-api-dom-subset). DOM APIs to add to host component instances.
* [Other maintenance](#other-maintenance). Misc tasks to reduce fragmentation.

## Implementation strategy

Where possible we should make these changes to the native code. However, we will prioritize user-space shims in JavaScript where they are faster to ship and allow us to gather feedback/support for this overall direction.

To further reduce risk, the underlying abstractions that we use to implement this proposed "Strict DOM/CSS" API should be flexible enough to support alternative APIs that can co-exist with (or replace) both what is proposed here as well as the existing React Native APIs.

## Adoption strategy

Many of these additions are already in progress and have been proven in a user-space shim at Meta. The features that cannot be shimmed are explicitly called out below. Those features require core changes to React Native. Other features can be shimmed in user-space JavaScript if desired (with any associated performance cost of doing so).

To encourage library and product developers to proactively migrate to these new APIs, React Native for Web plans to only support these W3C standards-based APIs in future versions. This will allow us to incrementally add APIs to React Native without needing to commit to simultaneously deprecating APIs and migrating existing React Native code. Existing React Native developers will adopt these APIs if they wish to support web.

## How we teach this

Teaching these APIs is simplified by developer familiarity with existing DOM / React DOM APIs.

## Drawbacks

Many of these features can be shimmed in user-space, at the cost of runtime overhead for native or web platforms. The runtime overhead for web is untenable. The runtime overhead of shifting shims into the React Native JS has yet to be established.

This adds redundancy to the React Native API, as the proposal is to make additive changes and avoid requiring removal of existing APIs in the short term.

We have to be very intentional about how much compatibility with open web standards we can promise. Some cross-platform limitations are rooted in significant differences in the host platforms APIs (e.g., accessibility) and the lack of 99% compat may frustrate users.

This proposal does not include concrete plans for how to support low-level navigation (i.e., `href`) on React Native. This is a crucial aspect of web accessibility and must be incorporated in a future revision.

-----

# Props API (React DOM subset)

One of the fastest and simplest ways to reduce fragmentation between React DOM and React Native is to support more React DOM props on the existing React Native components. This will help to reduce the network and runtime overhead of using React Native for Web, as it will be able to reduce the amount of code responsible for mapping React Native APIs to React DOM APIs. For developers familiar with React DOM, there will be less of a learning curve to applying that knowledge to React Native.

## Core components

The core components `View` and `Text`.

### Primitive props

Certain props common to all core components have direct equivalents in React DOM. We should support those props on native where possible and allow the others in the types definition. Supporting the ARIA 1.2 interface would allow more ARIA-targeting React DOM libraries to work on React Native, and would provide a well defined cross-platform target for equivalent native features (e.g., React Native for Windows needing heading levels, set size, etc.)

- [ ] [`autoFocus`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autofocus).
- [ ] [`aria-activedescendant`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-activedescendant).
- [ ] [`aria-atomic`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-atomic).
- [ ] [`aria-autocomplete`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-autocomplete).
- [x] [`aria-busy`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-busy) is the same as `accessibilityState.busy`.
- [x] [`aria-checked`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-checked) is the same as `accessibilityState.checked`.
- [ ] [`aria-colcount`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-colcount).
- [ ] [`aria-colindex`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-colindex).
- [ ] [`aria-colindextext`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-colindextext)
- [ ] [`aria-colspan`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-colspan).
- [ ] [`aria-controls`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-controls)
- [ ] [`aria-current`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-current)
- [ ] [`aria-describedby`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-describedby) is the same as [`accessibilityDescribedBy`](https://cs.github.com/microsoft/react-native-windows/blob/8ea39fa9737c81516e3cad495ed0e2ae939412ce/packages/@office-iss/react-native-win32/src/Libraries/Components/View/ViewWin32.Props.ts#L159-L165) (Desktop).
- [ ] ([`aria-description`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-description))
- [ ] [`aria-details`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-details).
- [x] [`aria-disabled`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-disabled) is the same as `accessibilityState.disabled`.
- [ ] [`aria-errormessage`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-errormessage) is similar to [`accessibilityErrorMessage`](https://github.com/facebook/react-native/pull/33468).
- [x] [`aria-expanded`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-expanded) is the same as `accessibilityState.expanded`.
- [ ] [`aria-flowto`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-flowto).
- [ ] [`aria-haspopup`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-haspopup).
- [x] [`aria-hidden`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-hidden) is the same as `accessibilityElementsHidden` (iOS) / `importantforAccessibility` (Android).
- [ ] [`aria-invalid`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-invalid).
- [ ] [`aria-keyshortcuts`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-keyshortcuts).
- [x] [`aria-label`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-label) is the same as `accessibilityLabel`.
- [x] [`aria-labelledby`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-labelledby) is the same as `accessibilityLabelledBy` (Android, Desktop).
- [ ] [`aria-level`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-level) is the same as [`accessibilityLevel`](https://github.com/microsoft/react-native-windows/blob/ab330f691477cad43c19b58ab8ce767a1c5baf58/packages/%40office-iss/react-native-win32/src/Libraries/Components/View/ViewWin32.Props.ts#L182-L186) (Desktop).
- [x] [`aria-live`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-live) is the same as `accessibilityLiveRegion` with `'off'` value changed to `'none'` (Android).
- [x] [`aria-modal`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-modal) is the same as `accessibilityViewIsModal` (iOS).
- [ ] [`aria-multiline`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-multiline).
- [ ] [`aria-multiselectable`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-multiselectable).
- [ ] [`aria-orientation`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-orientation).
- [ ] [`aria-owns`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-owns).
- [ ] [`aria-placeholder`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-placeholder).
- [ ] [`aria-posinset`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-posinset) is the same as [`accessibilityPosInSet`](https://github.com/microsoft/react-native-windows/blob/5dbd74b1852c38f851483478d6da7d81949d0195/vnext/src/Libraries/Components/View/ViewWindowsProps.ts#L42-L47) (Desktop)
- [ ] [`aria-pressed`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-pressed).
- [ ] [`aria-readonly`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-readonly).
- [ ] [`aria-required`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-required).
- [ ] [`aria-roledescription`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-roledescription).
- [ ] [`aria-rowcount`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-rowcount).
- [ ] [`aria-rowindex`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-rowindex).
- [ ] [`aria-rowindextext`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-rowindextext).
- [ ] [`aria-rowspan`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-rowspan).
- [x] [`aria-selected`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-selected) is the same as `accessibilityState.selected`.
- [ ] [`aria-setsize`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-setsize) is the same as [`accessibilitySetSize`](https://github.com/microsoft/react-native-windows/blob/5dbd74b1852c38f851483478d6da7d81949d0195/vnext/src/Libraries/Components/View/ViewWindowsProps.ts#L49-L54) (Desktop).
- [ ] [`aria-sort`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-sort).
- [x] [`aria-valuemax`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-valuemax) is the same as `accessibilityValue.max`.
- [x] [`aria-valuemin`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-valuemin) is the same as `accessibilityValue.min`.
- [x] [`aria-valuenow`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-valuenow) is the same as `accessibilityValue.now`.
- [x] [`aria-valuetext`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-valuetext) is the same as `accessibilityValue.text`.
- [ ] [`dir`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/dir) configures element's layout / writing direction.
- [ ] [`elementTiming`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/elementtiming).
- [ ] [`lang`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/lang) identifies the language of the text content.
- [ ] [`hidden`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/hidden).
- [x] [`id`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/id) is the same as `nativeID`.
- [ ] [`inert`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert).
- [x] [`role`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles) is the same as `accessibilityRole` (with support for web values).
- [x] `style` is unchanged.
- [x] [`tabIndex`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex) is the same as `focusable` (Android, Desktop). Add support for `0` and `-1` values only.

### Event props

Props for commonly handled events should also be supported on core components. Each event handler should receive a W3C event object of the corresponding type, not a synthetic event. These props are merely conveniences and otherwise identical to using the `EventTarget` API mentioned later on. Note that no capture phase props are included, as listening to the capture phase (as well as custom events) should be handled with the `EventTarget` API.

To avoid breaking changes in React Native and React DOM, it may be preferrable to create new event prop names that receive events as specified by web standards, e.g., `on-click`, `on-pointerup`.

- [ ] `onAuxClick` is a `PointerEvent` for auxillary clicks.
- [ ] `onBlur`.
- [ ] `onClick` is a `PointerEvent` for primary clicks.
- [ ] `onContextMenu`.
- [ ] `onCopy`.
- [ ] `onCut`.
- [ ] `onFocus`.
- [ ] `onGotPointerCapture`.
- [ ] `onLostPointerCapture`.
- [ ] `onPaste`.
- [ ] `onPointerCancel`.
- [ ] `onPointerDown`.
- [ ] `onPointerEnter`.
- [ ] `onPointerLeave`.
- [ ] `onPointerMove`.
- [ ] `onPointerOut`.
- [ ] `onPointerOver`.
- [ ] `onPointerUp`.
- [ ] `onKeyDown`.
- [ ] `onKeyUp`.

Example:

```jsx
<View
  aria-hidden={false}
  aria-label="Accessibility label"
  id="native-id"
  on-click={handleDOMClick}
  on-pointerdown={handlePointerDown}
  role="button"
  tabIndex={0}
>
```

## `Image` component

Inherits all props and events from "Core Component".

### Primitive props

Various React DOM `img` props have direct equivalents in React Native. For example, `srcSet` is a subset of the `source` prop but without support for specifying headers, etc. Features like alternative text content are also required to target web.

- [x] `alt` prop for alternative text support.
- [x] `crossOrigin` is equivalent to setting the relevant `Header` in the `source` object. Declaratively configure cross-origin permissions for loaded images.
- [ ] `decoding`.
- [ ] `fetchPriority`.
- [x] `height` is the same as `source.height`.
- [ ] `loading`.
- [x] `referrerPolicy` is equivalent to setting the relevant `Header` in the `source` object. Declaratively configure referrer policy.
- [x] `src` is the same as `source.uri`.
- [x] `srcSet` is the same as setting a `source` array with `uri` and `scale` properties defined.
- [x] `width` is the same as `source.width`.

Longer-term we should look to decouple the image component from the image loader. The latter could take the form of a module and/or hook to provide the props needed for a simpler image component as is found in React DOM.

### Event props

- [ ] `onError()`
- [ ] `onLoad()`

Example:

```jsx
<Image
  alt="Alternative text"
  crossOrigin="anonymous"
  srcSet="https://image.png 1x, https://image2.png 2x"
  width={320}
>
```

## `TextInput` component

Inherits all props and events from "Core Component".

### Primitive props

- [x] `autoComplete` is the same as mapped values for existing `autoComplete` (Android) and `textContentType` (iOS).
- [ ] `disabled`.
- [x] `enterKeyHint` is the same as mapped values for `returnKeyType`.
- [x] `inputMode`  is the same as mapped values for `keyboardType`.
  - [x] `inputMode === 'decimal'` is the same as `keyboardType = 'decimal-pad'`.
  - [x] `inputMode === 'email'` is the same as `keyboardType = 'email-address'`.
  - [x] `inputMode === 'none'` is the same as `showSoftInputOnFocus = false`.
  - [x] `inputMode === 'numeric'` is the same as `keyboardType = 'numeric'`.
  - [x] `inputMode === 'search'` is the same as `keyboardType = 'search'`.
  - [x] `inputMode === 'tel'` is the same as `keyboardType = 'phone-pad'`.
  - [x] `inputMode === 'url'` is the same as `keyboardType = 'url'`.
- [ ] `max`.
- [x] `maxLength`.
- [ ] `min`.
- [ ] `minLength`.
- [ ] `placeholder`.
- [x] `readOnly` is the same as inverse `editable`.
- [ ] `required`.
- [x] `rows` is the same as `numberOfLines`.
- [ ] `spellCheck`.
- [ ] `value`.

### Event props

- [ ] `onBeforeInput()`
- [ ] `onChange()`
- [ ] `onInput()`
- [ ] `onInvalid()`
- [ ] `onSelect()`
- [ ] `onSelectionChange()`

Example:

```jsx
<TextArea
  autoComplete="email"
  inputMode="email"
/>

<TextArea
  multiline
  readOnly
  rows={3}
/>
```

## Appendix

`autoComplete` mapping:

```js
// https://reactnative.dev/docs/textinput#autocomplete-android
const autoCompleteAndroid = {
  'address-line1': 'postal-address-region',
  'address-line2': 'postal-address-locality',
  bday: 'birthdate-full',
  'bday-day': 'birthdate-day',
  'bday-month': 'birthdate-month',
  'bday-year': 'birthdate-year',
  'cc-csc': 'cc-csc',
  'cc-exp': 'cc-exp',
  'cc-exp-month': 'cc-exp-month',
  'cc-exp-year': 'cc-exp-year',
  'cc-number': 'cc-number',
  country: 'postal-address-country',
  'current-password': 'password',
  email: 'email',
  name: 'name',
  'additional-name': 'name-middle',
  'family-name': 'name-family',
  'given-name': 'name-given',
  'honorific-prefix': 'namePrefix',
  'honorific-suffix': 'nameSuffix',
  'new-password': 'password-new',
  off: 'off',
  'one-time-code': 'sms-otp',
  'postal-code': 'postal-code',
  sex: 'gender',
  'street-address': 'street-address',
  tel: 'tel',
  'tel-country-code': 'tel-country-code',
  'tel-national': 'tel-national',
  username: 'username'
};

// https://reactnative.dev/docs/textinput#textcontenttype-ios
const autoCompleteIOS = {
  'address-line1': 'streetAddressLine1',
  'address-line2': 'streetAddressLine2',
  'cc-number': 'creditCardNumber',
  'current-password': 'password',
  country: 'countryName',
  email: 'emailAddress',
  name: 'name',
  'additional-name': 'middleName',
  'family-name': 'familyName',
  'given-name': 'givenName',
  nickname: 'nickname',
  'honorific-prefix': 'name-prefix',
  'honorific-suffix': 'name-suffix',
  'new-password': 'newPassword',
  off: 'none',
  'one-time-code': 'oneTimeCode',
  organization: 'organizationName',
  'organization-title': 'jobTitle',
  'postal-code': 'postalCode',
  'street-address': 'fullStreetAddress',
  tel: 'telephoneNumber',
  url: 'URL',
  username: 'username'
};
```

`srcSet` to `source` mapping:

```js
const { crossOrigin, referrerPolicy, srcSet } = props;
const source = [];
const srcList = srcSet.split(', ');
srcList.forEach((src) => {
  const [uri, xscale] = src.split(' ');
  const scale = parseInt(xscale.split('x')[0], 10);
  const headers = {};
  if (crossOrigin === 'use-credentials') {
    headers['Access-Control-Allow-Credentials'] = true;
  }
  if (referrerPolicy != null) {
    headers['Referrer-Policy'] = referrerPolicy;
  }
  source.push({ headers, height, scale, uri, width });
});
```

-----

# Styles API (CSS subset)

The styling capabilities of React Native should be significantly expanded to cover more of the features that are heavily relied upon by web engineers. Furthermore, React Native needs to gracefully handle unknown style properties or values as is expected on the web. Rather than hard crashing it could print warnings to bring attention to any styles that are not currently supported on native.

## StyleSheet

* [ ] Deprecate `StyleSheet.absoluteFill`. Use `position:'absolute';inset:0;` instead.
* [ ] Deprecate `StyleSheet.absoluteFillObject`.
* [ ] Deprecate `StyleSheet.compose()`. Use existing `[ a, b ]` syntax instead.
* [ ] Deprecate `StyleSheet.flatten()`. This API encourages runtime introspection of styles in render calls. This pattern is a performance cost (flattening arrays of objects), and prevents us from supporting build-time optimizations for web (e.g., extracting styles to CSS files).
* [ ] Deprecate or rename `setStyleAttributePreprocessor` (e.g., `unstable_setStyleAttributePreprocessor`).
* [ ] `StyleSheet.create()` should obfuscate styles to prevent introspection (i.e., revert identify function change). On web, we need to be able to remove the JavaScript style objects from bundles to support build-time optimization like extraction to CSS files.
* [ ] `StyleSheet` should be a function that returns native props, e.g., `const nativeProps = StyleSheet(...styles)`. On web, this is used to pass DOM `className` and `style` props, whereas on native it can be used to pass flattened/transformed styles and any other other native props to native components.

## CSS Compatibility

Existing properties that can be adjusted to align with the CSS spec.

* [x] `aspectRatio`. Support string values, i.e., `'16 / 9'`, to align with CSS.
* [x] `fontVariant` support space-separated string values to align with CSS.
* [x] `fontWeight` support number values to align with React DOM / CSS.
* [x] `objectFit` is equivalent to `resizeMode` for `<Image>`.
* [x] `pointerEvents` is equivalent to `pointerEvents` prop.
* [ ] `position`. Support for `fixed` and `sticky` values.
* [x] `transform`. Support using string values to set transforms.
* [x] `verticalAlign` is equivalent to `textAlignVertical`.
* [x] `userSelect`. Equivalent to using `selectable` prop on `<Text>`.

Existing logical properties that can be adjusted to adopt the CSS standard names. In addition, React Native will need to add native support for subtree-level writing direction controls. Setting the `dir` prop (or `direction` style) to `ltr` or `rtl` on an element should alter the way logical properties are resolved in the subtree.

* [ ] `direction` is equivalent to `writingDirection`.
* [ ] `borderEndEndRadius` is equivalent to `borderBottomEndRadius`.
* [ ] `borderEndStartRadius` is equivalent to `borderBottomStartRadius`.
* [ ] `borderStartEndRadius` is equivalent to `borderTopEndRadius`.
* [ ] `borderStartStartRadius` is equivalent to `borderTopStartRadius`.
* [ ] `borderBlockColor` is equivalent to `borderTopColor` & `borderBottomColor`.
* [ ] `borderBlockEndColor` is equivalent to `borderBottomColor`.
* [ ] `borderBlockStartColor` is equivalent to `borderTopColor`.
* [ ] `borderInlineColor` is equivalent to `borderEndColor` & `borderStartColor`.
* [ ] `borderInlineEndColor` is equivalent to `borderEndColor`.
* [ ] `borderInlineStartColor` is equivalent to `borderStartColor`.
* [ ] `borderBlockStyle` is equivalent to `borderTopStyle` & `borderBottomStyle`.
* [ ] `borderBlockEndStyle` is equivalent to `borderBottomStyle`.
* [ ] `borderBlockStartStyle` is equivalent to `borderTopStyle`.
* [ ] `borderInlineStyle` is equivalent to `borderEndStyle` & `borderStartStyle`.
* [ ] `borderInlineEndStyle` is equivalent to `borderEndStyle`.
* [ ] `borderInlineStartStyle` is equivalent to `borderStartStyle`.
* [ ] `borderBlockWidth` is equivalent to `borderTopWidth` & `borderBottomWidth`.
* [ ] `borderBlockEndWidth` is equivalent to `borderBottomWidth`.
* [ ] `borderBlockStartWidth` is equivalent to `borderTopWidth`.
* [ ] `borderInlineWidth` is equivalent to `borderEndWidth` & `borderStartWidth`.
* [ ] `borderInlineEndWidth` is equivalent to `borderEndWidth`.
* [ ] `borderInlineStartWidth` is equivalent to `borderStartWidth`.
* [x] `marginInlineStart` is equivalent to `marginStart`.
* [x] `marginInlineEnd` is equivalent to `marginEnd`.
* [x] `marginBlockStart` is equivalent to `marginTop`.
* [x] `marginBlockEnd` is equivalent to `marginBottom`.
* [x] `marginBlock` is equivalent to `marginVertical`.
* [x] `marginInline` is equivalent to `marginHorizontal`.
* [x] `paddingInlineStart` is equivalent to `paddingStart`.
* [x] `paddingInlineEnd` is equivalent to `paddingEnd`.
* [x] `paddingBlockStart` is equivalent to `paddingTop`.
* [x] `paddingBlockEnd` is equivalent to `paddingBottom`.
* [x] `paddingBlock` is equivalent to `paddingVertical`.
* [x] `paddingInline` is equivalent to `paddingHorizontal`.
* [ ] `inset` is equivalent to `top` & `bottom` & `right` & `left`.
* [ ] `insetBlock` is equivalent to `top` & `bottom`.
* [ ] `insetBlockEnd` is equivalent to `bottom`.
* [ ] `insetBlockStart` is equivalent to `top`.
* [ ] `insetInline` is equivalent to `right` & `left`.
* [ ] `insetInlineEnd` is equivalent to `right` or `left`.
* [ ] `insetInlineStart` is equivalent to `right` or `left`.

## CSS Animations

Support declarative keyframes and animations that can be optimized on the native side and avoid the need for `Animated`. Consider dispatching the corresponding W3C animation events too. See [animation](https://developer.mozilla.org/en-US/docs/Web/CSS/animation)).

## CSS Colors

Support CSS 4 Colors using [Colorjs.io](https://colorjs.io/).

## CSS Container Queries

Prepare for [CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries).

## CSS Custom Properties

Support CSS custom property syntax `--variable-name`. This could be shimmed in user-space on top of the existing `StyleSheet` API, with a React Context used to provide variables and values to a subtree.

## CSS Filters

Support declarative filters as used by the CSS [filter](https://developer.mozilla.org/en-US/docs/Web/CSS/filter) style.

## CSS Functions

* [ ] `calc()`(https://developer.mozilla.org/en-US/docs/Web/CSS/calc)
* [ ] `clamp()`(https://developer.mozilla.org/en-US/docs/Web/CSS/clamp)
* [ ] `max()`(https://developer.mozilla.org/en-US/docs/Web/CSS/max)
* [ ] `min()`(https://developer.mozilla.org/en-US/docs/Web/CSS/min)
* [ ] `minmax()`(https://developer.mozilla.org/en-US/docs/Web/CSS/minmax)
* [ ] `url()`(https://developer.mozilla.org/en-US/docs/Web/CSS/url)
* [ ] `var()`(https://developer.mozilla.org/en-US/docs/Web/CSS/var)

## CSS Lengths

Support major units where supported by CSS.

* [ ] `em` units.
* [ ] `rem` units.
* [ ] `px` units.
* [ ] `v*` units.
* [ ] `%` units.

## CSS Media Queries

Support CSS Media Queries. Although Media Queries are not a preferred long-term solution, [Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries) are not yet widely supported by browsers. This could be shimmed in user-space on top of the existing `StyleSheet` API.

## CSS Properties and Values

Miscellaneous CSS properties and values that should be supported on native.

* [ ] `backgroundImage`. Add support for setting background images via `url()`. Stretch: support CSS gradients.
* [ ] `boxSizing`.
* [ ] `boxShadow`. Add native support for CSS box shadows to replace buggy, iOS-specific `shadow*` styles.
* [ ] `caretColor`.
* [ ] `clipPath`.
* [ ] `display` values of `block`, `inline`, `inline-block`, `inline-flex`.
* [ ] `float`.
* [ ] [`justifySelf`](https://developer.mozilla.org/en-US/docs/Web/CSS/justify-self). Sets the way a box is justified inside its alignment container along the appropriate axis.
* [ ] (`lineClamp` is equivalent to `numberOfLines` prop on `<Text>` components).
* [ ] [`objectPosition`](https://developer.mozilla.org/en-US/docs/Web/CSS/object-position).
* [ ] `overflowBlock`.
* [ ] `overflowInline`.
* [ ] `overflowX`.
* [ ] `overflowY`.
* [ ] [`placeContent`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-content). Shorthand for setting `alignContent` and `justifyContent`. Value is an enum of `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `right`, `safe`, `space-around`, `space-between`, `space-evenly`, `start`, `stretch`, `unsafe`. Note that `space-evenly` should also be supported for `alignContent` and `justifyContent`.
* [ ] [`placeItems`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-items). Shorthand for setting `alignItems` and `justifyItems`. Value is an enum of `auto`, `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `right`, `start`, `stretch`.
* [ ] [`placeSelf`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-self). Shorthand for setting `alignSelf` and `justifySelf`. Value is an enum of `auto`, `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `normal`, `right`, `start`, `stretch`.
* [ ] [`rotate`](https://developer.mozilla.org/en-US/docs/Web/CSS/rotate).
* [ ] [`scale`](https://developer.mozilla.org/en-US/docs/Web/CSS/scale).
* [ ] `textShadow`. Add native support for CSS text shadows.
* [ ] [`touchAction`](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action).
* [ ] [`translate`](https://developer.mozilla.org/en-US/docs/Web/CSS/translate).
* [ ] `visibility`. Add support for visually hiding elements.
* [ ] [`whiteSpace`](https://developer.mozilla.org/en-US/docs/Web/CSS/white-space).

## CSS Transitions

Support declarative transitions that can be optimized on the native side and avoid the need for `Animated`. Support `transitionProperty`, `transitionDuration`, `transitionTimingFunction`, and `transitionDelay` styles (see [transition](https://developer.mozilla.org/en-US/docs/Web/CSS/transition)). Consider dispatching the corresponding W3C transition events too.

-----

# Environment API

## document

Support the `document` object for common patterns such as listening to document resize, listening to capture or bubble phase events for the entire application (e.g., "outside click" pattern), etc.

### Properties

- [ ] [`activeElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/activeElement).
- [ ] [`visibilityState`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilityState)

### Events

* [ ] `scroll`.
* [ ] `visibilitychange`.

## IntersectionObserver

Support the [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver) API for observing the intersection of target elements. This can also be used as a building block for performance tooling.

## MutationObserver

Support the latest [`MutationObserver`](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) API for watching changes to the host node tree. This can also be used as a building block for performance tooling.

## ResizeObserver

Support the latest [`ResizeObserver`](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver) API for responding the changes in the size of elements. Note that this API no longer includes positional coordinates and is optimized for width/height information.

-----

# Components API (DOM subset)

A logical extension of support more React DOM props is to support a subset of React DOM components directly.

This would provide a built-in declarative API for cross-platform elements in React Native. All unknown tags default to `<span>` layout. All known-but-unimplemented block-level tags default to `<div>` layout. Most tags can be mapped to existing React Native components and props, in the reverse of the [mapping used by React Native for Web](https://github.com/necolas/react-native-web/blob/b2eb3ca03ffc4b8570591b6c2082c6e4edae0e7f/packages/react-native-web/src/modules/AccessibilityUtil/propsToAccessibilityComponent.js#L12-L30).

## Sections

Content sectioning elements allow you to organize the document content into logical pieces. Use the sectioning elements to create a broad outline for your page content, including header and footer navigation, and heading elements to identify sections of content.

* [ ] `article` is equivalent to `View` with `display:block` layout and `role="article"`.
* [ ] `aside`
* [ ] `div` is equivalent to `View` with `display:block` layout.
* [ ] `footer`.
* [ ] `h1-6`.
* [ ] `header`.
* [ ] `main`.
* [ ] `nav`.
* [ ] `section` is equivalent to `View` with `display:block` layout and `role="section"`.

## Text

Text content elements organize blocks or sections of content. Important for accessibility, these elements identify the purpose or structure of that content.

* [ ] `blockquote`.
* [ ] `li`.
* [ ] `ol`.
* [ ] `ul`.
* [ ] `p`.
* [ ] `pre`.

## Inline text

Inline text semantics define the meaning, structure, or style of a word, line, or any arbitrary piece of text.

* [ ] `a`.
* [ ] `b`.
* [ ] `bdi`.
* [ ] `bdo`.
* [ ] `code`.
* [ ] `em`.
* [ ] `span` is equivalent to `Text` with `display:inline` layout.
* [ ] `strong`.
* [ ] `sub`.
* [ ] `sup`.

## Media

* [ ] `img` is equivalent to `Image` without advanced loader configuration.

## Forms

Elements which can be used together to create forms which the user can fill out and submit. There's a great deal of further information about this available in the [HTML forms guide](https://developer.mozilla.org/en-US/docs/Learn/Forms).

* [ ] `button`.
* [ ] `input` is equivalent to `TextInput` with `inputMode` corresponding to `type`.
* [ ] `label`.
* [ ] `optgroup`.
* [ ] `option`.
* [ ] `progress`.
* [ ] `select`.
* [ ] `textarea` is equivalent to `TextInput` with `multiline={true}`.

-----

# Elements API (DOM subset)

Subset of cross-platform DOM APIs exposed on element instances, document, etc. This would be an imperative API for working with the Shadow tree in React Native. Based on data from [MDN Web API](https://developer.mozilla.org/en-US/docs/Web/API), please refer to the MDN links for complete DOM APIs and comment if you believe any should be included in the proposed subset.

In practice, this DOM API subset is generally limited to the “best practice” APIs that can safely be used with React DOM (i.e., read-only operations that don’t modify the DOM tree), and most React components should already be limited to this subset.

## EventTarget

The `EventTarget` API is implemented in JS environments, and should be available on React Native host elements. This feature is commonly used on web, and could be used by React Native developers to support more complex features without first requiring further core API changes to support each use case.

### Methods

- [ ] `EventTarget.addEventListener()`. Registers an event handler to a specific event type on the element.
- [ ] `EventTarget.dispatchEvent()`. Dispatches an event to this node in the DOM and returns a boolean value that indicates whether no handler canceled the event.
- [ ] `EventTarget.removeEventListener()`. Removes an event listener from the element.

## Node

The DOM Node interface is an abstract base class upon which many other DOM API objects are based, thus letting those object types to be used similarly and often interchangeably. As an abstract class, there is no such thing as a plain Node object. All objects that implement Node functionality are based on one of its subclasses. Every kind of host node is represented by an interface based on Node.

In some cases, a particular feature of the base Node interface may not apply to one of its child interfaces; in that case, the inheriting node may return null or throw an exception, depending on circumstances.

### Methods

In addition to the properties below, Node inherits methods from its parent, EventTarget.

- [ ] `Node.contains()`. Returns true or false value indicating whether or not a node is a descendant of the calling node.
- [ ] `Node.getRootNode()`. Returns the context object's root which optionally includes the shadow root if it is available..

## Element

Element is the most general base class from which all element objects (i.e. objects that represent elements) in a Document inherit. It only has methods and properties common to all kinds of elements. More specific classes inherit from Element.

### Properties

Element inherits properties from its parent interface, Node, and by extension that interface's parent, EventTarget.

- [ ] `Element.clientHeight` [Read only]. Returns a number representing the inner height of the element.
- [ ] `Element.clientLeft` [Read only]. Returns a number representing the width of the left border of the element.
- [ ] `Element.clientTop` [Read only]. Returns a number representing the width of the top border of the element.
- [ ] `Element.clientWidth` [Read only]. Returns a number representing the inner width of the element.
- [ ] `Element.id` [Read only]. Is a DOMString representing the id of the element.
- [ ] `Element.scrollHeight` [Read only]. Returns a number representing the scroll view height of an element.
- [ ] `Element.scrollLeft`. Is a number representing the left scroll offset of the element.
- [ ] `Element.scrollLeftMax` [Read only]. Returns a number representing the maximum left scroll offset possible for the element.
- [ ] `Element.scrollTop`. A number representing number of pixels the top of the element is scrolled vertically.
- [ ] `Element.scrollTopMax` [Read only]. Returns a number representing the maximum top scroll offset possible for the element.
- [ ] `Element.scrollWidth` [Read only]. Returns a number representing the scroll view width of the element.

### Methods

Element inherits methods from its parents Node, and its own parent, EventTarget.

- [ ] `Element.computedStyleMap()`. Returns a StylePropertyMapReadOnly interface which provides a read-only representation of a CSS declaration block that is an alternative to CSSStyleDeclaration.
- [ ] `Element.getBoundingClientRect()` **P1**. Returns the size of an element and its position relative to the viewport.
- [ ] `Element.scroll()`. Scrolls to a particular set of coordinates inside a given element.
- [ ] `Element.scrollBy()`. Scrolls an element by the given amount.
- [ ] `Element.scrollIntoView()`. Scrolls the page until the element gets into the view.
- [ ] `Element.scrollTo()`. Scrolls to a particular set of coordinates inside a given element.

### Events

Listen to these events using `addEventListener()` or by assigning an event handler to the equivalent component prop.

- [ ] `error`. Fired when a resource failed to load, or can't be used. For example, if a script has an execution error or an image can't be found or is invalid. Also available via the `onError` property.
- [ ] `scroll`. Fired when the document view or an element has been scrolled. Also available via the `onScroll` property.
- [ ] `select`. Fired when some text has been selected. Also available via the `onSelect` property.
- [ ] `wheel`. Fired when the user rotates a wheel button on a pointing device (typically a mouse). Also available via the `onWheel` property.

Clipboard events

- [ ] `copy`. Fired when the user initiates a copy action through the browser's user interface. Also available via the `onCopy` property.
- [ ] `cut`. Fired when the user initiates a cut action through the browser's user interface. Also available via the `onCut` prop.
- [ ] `paste`. Fired when the user initiates a paste action through the browser's user interface. Also available via the `onPaste` prop.

Focus events

- [ ] `blur`. Fired when an element has lost focus. Also available via the `onBlur` prop.
- [ ] `focus`. Fired when an element has gained focus. Also available via the `onFocus` prop.
- [ ] `focusin`. Fired when an element is about to gain focus.
- [ ] `focusout`. Fired when an element is about to lose focus.

Keyboard events

- [ ] `keydown`. Fired when a key is pressed. Also available via the `onKeyDown` prop.
- [ ] `keyup`. Fired when a key is released. Also available via the `onKeyUp` prop.

Pointer events

- [ ] `auxclick`. Fired when a non-primary pointing device button (e.g., any mouse button other than the left button) has been pressed and released on an element. Also available via the `onAuxClick` prop.
- [ ] `click`. Fired when a pointing device button (e.g., a mouse's primary button) is pressed and released on a single element. Also available via the `onClick` prop.
- [ ] `contextmenu`. Fired when the user attempts to open a context menu. Also available via the oncontextmenu property.

## HTMLElement

The HTMLElement interface represents any HTML element. Some elements directly implement this interface, while others implement it via an interface that inherits it.

### Properties

Inherits properties from its parent, Element, and implements those from DocumentAndElementEventHandlers, GlobalEventHandlers, and TouchEventHandlers.

- [ ] `HTMLElement.hidden` [Read only]. A boolean value indicating if the element is hidden or not.
- [ ] `HTMLElement.offsetHeight` [Read only]. Returns a double containing the height of an element, relative to the layout.
- [ ] `HTMLElement.offsetLeft` [Read only]. Returns a double, the distance from this element's left border to its offsetParent's left border.
- [ ] `HTMLElement.offsetParent` [Read only]. Returns a Element that is the element from which all offset calculations are currently computed.
- [ ] `HTMLElement.offsetTop` [Read only]. Returns a double, the distance from this element's top border to its offsetParent's top border.
- [ ] `HTMLElement.offsetWidth` [Read only]. Returns a double containing the width of an element, relative to the layout.

### Methods

Inherits methods from its parent, Element, and implements those from DocumentAndElementEventHandlers, GlobalEventHandlers, and TouchEventHandlers.

- [x] `HTMLElement.blur()`. Removes keyboard focus from the currently focused element.
- [ ] `HTMLElement.click()`. Sends a mouse click event to the element.
- [ ] `HTMLElement.focus(options)`. Makes the element the current keyboard focus.

### Events

Listen to these events using `addEventListener()` or by assigning an event listener to the equivalent component prop.

- [ ] `invalid`. Fired when an element does not satisfy its constraints during constraint validation. Also available via the `onInvalid` prop.

Input events

- [ ] `beforeinput`. Fired when the value of an `<input>`, `<select>`, or `<textarea>` element is about to be modified.
- [ ] `input`. Fired when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed. Also available via the `onInput` property.
- [ ] `change`. Fired when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed and committed by the user. Unlike the input event, the change event is not necessarily fired for each alteration to an element's value. Also available via the `onChange` property.

Pointer events

- [ ] `gotpointercapture`. Fired when an element captures a pointer using `setPointerCapture()`. Also available via the `onGotPointerCapture` prop.
- [ ] `lostpointercapture`. Fired when a captured pointer is released. Also available via the `onLostPointerCapture` prop.
- [ ] `pointercancel`. Fired when a pointer event is canceled. Also available via the onpointercancel property.
- [ ] `pointerdown`. Fired when a pointer becomes active. Also available via the onpointerdown property.
- [ ] `pointerenter`. Fired when a pointer is moved into the hit test boundaries of an element or one of its descendants. Also available via the onpointerenter property.
- [ ] `pointerleave`. Fired when a pointer is moved out of the hit test boundaries of an element. Also available via the onpointerleave property.
- [ ] `pointermove`. Fired when a pointer changes coordinates. Also available via the onpointermove property.
- [ ] `pointerout`. Fired when a pointer is moved out of the hit test boundaries of an element (among other reasons). Also available via the `onPointerOut` prop.
- [ ] `pointerover`. Fired when a pointer is moved into an element's hit test boundaries. Also available via the `onPointerOver` prop.
- [ ] `pointerup`. Fired when a pointer is no longer active. Also available via the `onPointerUp` prop.

## HTMLImageElement

The HTMLImageElement interface represents an HTML `<img>` element, providing the properties and methods used to manipulate image elements.

### Properties

Inherits properties from its parent, HTMLElement.

- [ ] `HTMLImageElement.complete` [Read only]. Returns a boolean value that is true if the browser has finished fetching the image, whether successful or not. That means this value is also true if the image has no src value indicating an image to load.
- [ ] `HTMLImageElement.currentSrc` [Read only]. Returns a USVString representing the URL from which the currently displayed image was loaded. This may change as the image is adjusted due to changing conditions, as directed by any media queries which are in place.
- [ ] `HTMLImageElement.naturalHeight` [Read only]. Returns an integer value representing the intrinsic height of the image in CSS pixels, if it is available; else, it shows 0. This is the height the image would be if it were rendered at its natural full size.
- [ ] `HTMLImageElement.naturalWidth` [Read only]. An integer value representing the intrinsic width of the image in CSS pixels, if it is available; otherwise, it will show 0. This is the width the image would be if it were rendered at its natural full size.

### Events

- [ ] `error`. Also available via the `onError` prop.
- [ ] `load`. Also available via the `onLoad` prop.

## HTMLInputElement

The HTMLInputElement interface provides special properties and methods for manipulating the options, layout, and presentation of `<input>` elements.

### Properties

Properties that apply only to visible elements containing text or numbers

- [ ] `selectionEnd`. unsigned long: Returns / Sets the end index of the selected text. When there's no selection, this returns the offset of the character immediately following the current text input cursor position.
- [ ] `selectionStart`. unsigned long: Returns / Sets the beginning index of the selected text. When nothing is selected, this returns the position of the text input cursor (caret) inside of the <input> element.
- [ ] `selectionDirection`. string: Returns / Sets the direction in which selection occurred. Possible values are: forward (the selection was performed in the start-to-end direction of the current locale), backward (the opposite direction) or none (the direction is unknown).
- [ ] `value`. string: Returns / Sets the current value of the control. If the user enters a value different from the value expected, this may return an empty string.

### Methods

- [ ] `select()`. Selects all the text in the input element, and focuses it so the user can subsequently replace all of its content.
- [ ] `setSelectionRange()`. Selects a range of text in the input element (but does not focus it).
- [ ] `showPicker()`. Shows a browser picker for date, time, color, and files.

### Events

Listen to these events using `addEventListener()` or by assigning an event listener to the oneventname property of this interface:

- [ ] `input`. Fires when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed. Note that this is actually fired on the HTMLElement interface and also applies to contenteditable elements, but we've listed it here because it is most commonly used with form input elements. Also available via the `onInput` prop.
- [ ] `invalid`. Fired when an element does not satisfy its constraints during constraint validation. Also available via the `onInvalid` prop.
- [ ] `selectionchange` event **P1**. Fires when the text selection in a <input> element has been changed. Also available via the `onSelectionChange` prop.

## HTMLTextAreaElement

The HTMLInputElement interface provides special properties and methods for manipulating the options, layout, and presentation of `<input>` elements.

### Properties

- [ ] `selectionEnd`. unsigned long: Returns / Sets the end index of the selected text. When there's no selection, this returns the offset of the character immediately following the current text input cursor position.
- [ ] `selectionStart`. unsigned long: Returns / Sets the beginning index of the selected text. When nothing is selected, this returns the position of the text input cursor (caret) inside of the `<textarea>` element.
- [ ] `selectionDirection`. string: Returns / Sets the direction in which selection occurred. Possible values are: forward (the selection was performed in the start-to-end direction of the current locale), backward (the opposite direction) or none (the direction is unknown).
- [ ] `value`. string: Returns / Sets the current value of the control. If the user enters a value different from the value expected, this may return an empty string.

### Methods

- [ ] `select()`. Selects all the text in the input element, and focuses it so the user can subsequently replace all of its content.
- [ ] `setSelectionRange()`. Selects a range of text in the input element (but does not focus it).

### Events

Listen to these events using `addEventListener()` or using the equivalent prop name.

- [ ] `input`. Fires when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed.  Also available via the `onInput` prop.
- [ ] `selectionchange`. Fires when the text selection in a `<textarea>` element has been changed.  Also available via the `onSelectionChange` prop.

-----

# Other maintenance

Other tasks that will help to reduce API fragmentation within React Native, and between React Native and React Native for Web. Large, user-space modules currently exported from the `react-native` package could eventually be moved entirely to separate packages, reducing the size of the core library on native and web, and allowing alternative solutions to compete on a level playing field. JavaScript from the React Native ecosystem should not be published to npm without removal of non-standard syntax.

* [ ] Deprecate the `Touchable*` components. Use `Pressable`.
* [ ] Add `hovered` state support to `Pressable`.
* [ ] Add `focused` state support to `Pressable`.
* [ ] Move `Animated` out of core and into a separate package in the monorepo.
* [ ] Move `VirtualizedList` out of core and into a separate package in the monorepo.
* [ ] Move `VirtualizedSectionList` out of core and into a separate package in the monorepo.
* [ ] Move `FlatList` out of core and into a separate package in the monorepo.
* [ ] Move `SectionList` out of core and into a separate package in the monorepo.
* [ ] Strip flow types and JSX from pacakges published to npm (make this the default for the community).
* [ ] Prevent libraries importing `'react-native'` package internals, by using the Node.js package exports API.
