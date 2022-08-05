---
title: Reduce API fragmentation across platforms
author:
- Nicolas Gallagher
date: 2021-08-03
---

# RFC: Reduce API fragmentation across platforms

## Summary

This is a proposal to incrementally reduce the API fragmentation faced by developers using React Native to target multiple platforms, and sharing code between native and web. Each proposed change is additive and can be tackled relatively independently. The aim is to avoid needing to migrate existing React Native code, and to make deprecations an optional and future part of this work.

## Motivation

React Native currently includes many APIs that are modelled on Web APIs but do not conform to the standards of those Web APIs. React Native also includes many APIs that achieve the same results on Android and iOS but are exposed as 2 different props. And React Native includes several APIs that have known performance (network and runtime) drawbacks on Web.

This proposal aims to allow developers to target more platforms with cross-platform APIs, and deliver better performance when targeting browsers. Features for Android, iOS, and Web are unified by aligning with the Web standard to a) minimize the overhead when running in browsers, and b) reduce developer education required to learn these features.

## Detailed design

Many of these additions are already in progress and have been proven in a user-space shim at Meta. The features that cannot be shimmed are explicitly called out below. Those features require core changes to React Native. Other features can be shimmed in user-space JavaScript if desired (with any associated performance cost of doing so).

### New common props

Certain props common to all core components have direct equivalents in React DOM. We should support those props where possible. For example, many of the existing `accessibility*` props in React Native are currently mapped to `aria-*` equivalents by React Native for Web. Supporting the ARIA interface would allow more ARIA-targeting React DOM libraries to work on React Native.

* ARIA props
  * `aria-busy` is the same as `accessibilityState.busy`.
  * `aria-checked` is the same as `accessibilityState.checked`.
  * `aria-disabled` is the same as `accessibilityState.disabled`.
  * `aria-expanded` is the same as `accessibilityState.expanded`.
  * `aria-hidden` is the same as `accessibilityElementsHidden` (iOS) / `importantforAccessibility` (Android).
  * `aria-label` is the same as `accessibilityLabel`.
  * `aria-live` is the same as `accessibilityLiveRegion` (with `'off'` value changed to `'none'`).
  * `aria-modal` is the same as `accessibilityViewIsModal`.
  * `aria-selected` is the same as `accessibilityState.selected`.
  * `aria-valuemax` is the same as `accessibilityValue.max`.
  * `aria-valuemin` is the same as `accessibilityValue.min`.
  * `aria-valuenow` is the same as `accessibilityValue.now`.
  * `aria-valuetext` is the same as `accessibilityValue.text`.
  * `role` is the same as `accessibilityRole` (with support for web values).
  * See prior work: [RFC Accessibility APIs](https://github.com/react-native-community/discussions-and-proposals/pull/410/files).
* Add `dir` prop to configure element's layout / writing direction.
* Make `id` the same as `nativeID`.
* Make `onAuxClick` the same as React DOM `onAuxClick` PointerEvent (can't be shimmed).
* Make `onClick` the same as React DOM `onClick` as PointerEvent (can't be shimmed).
* Make `onPointer*` props that same as React DOM PointerEvent props (can't be shimmed).
  * The Android and iOS implementation is currently being developed by Meta.
* Make `tabIndex` the same as React Native `focusable` prop.
  * Add support for `0` and `-1` values only.

Example:

```jsx
<View
  aria-hidden={false}
  aria-label="Accessibility label"
  id="native-id"
  onClick={handleDOMClick}
  onPointerDown={handlePointerDown}
  role="button"
  tabIndex={0}
>
```

### New props for `<Image>`

Various React DOM `img` props also have direct equivalents in React Native. For example, `srcSet` is a subset of the `source` prop. Features like alternative text content are also required to target web.

* Add `alt` prop for alternative text support.
* Add `crossOrigin` prop to declaratively configure cross-origin permissions for loaded images. Equivalent to setting the relevant `Header` in the `source` object.
* Add `referrerPolicy` to declaratively configure referrer policy. Equivalent to setting the relevant `Header` in the `source` object.
* Make `src` that same as setting `source.uri`.
* Make `srcSet` the same as setting a `source` array with `uri` and `scale` properties defined.
* Add `tintColor` prop.
  * Image currently sets `tintColor` via the inclusion of a non-standard style property.
  * Recommend adding `tintColor` prop to replace the React Native style property, to avoid the need to parse out `tintColor` from style for platforms like web.

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

Example:

```jsx
<Image
  alt="Alternative text"
  crossOrigin="anonymous"
  srcSet="https://image.png 1x, https://image2.png 2x"
  tintColor="purple"
>
```

### New props for `<TextInput>`

* Redefine `autoComplete` prop.
  * Unify values for Android (`autoComplete`) and iOS (`textContentType`) with Web (`autoComplete`).
  * See below for example mapping
* Add `enterKeyHint` prop and map to equivalent `returnKeyType` values.
* Add `inputMode` prop and map to equivalent `keyboardType` values.
  * `inputMode === 'decimal'` is the same as `keyboardType = 'decimal-pad'`
  * `inputMode === 'email'` is the same as `keyboardType = 'email-address'`
  * `inputMode === 'numeric'` is the same as `keyboardType = 'numeric'`
  * `inputMode === 'search'` is the same as `keyboardType = 'search'`
  * `inputMode === 'tel'` is the same as `keyboardType = 'phone-pad'`
  * `inputMode === 'url'` is the same as `keyboardType = 'url'`
* Make `readOnly` the same as `editable` (inverse).
* Make `rows` the same as `numberOfLines`.
  * Support auto-expanding textarea on native platforms.

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

### New styles

* CSS Custom Properties
  * Support CSS custom property syntax `--variable-name`.
  * This can be shimmed in user-space on top of the existing `StyleSheet` API, with a React Context used to provide variables and values to a subtree.
* CSS Lengths
  * Support `em`, `rem`, `v*`, `px` length units.
  * This can be shimmed in user-space on top of the existing `StyleSheet` API, with a React Context used to provide ancestral font-size information for `rem` values.
* CSS Colors
  * Support CSS Colors using [Colorjs.io](https://colorjs.io/).
  * This can be shimmed in user-space.
* CSS Media Queries
  * Support CSS Media Queries. Although Media Queries are not a preferred long-term solution, [Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries) are not yet widely supported by browsers.
  * This can be shimmed in user-space on top of the existing `StyleSheet` API.
* CSS Logical Properties
  * Support CSS logical property names, i.e., `paddingInlineStart` rather than only non-standard `paddingStart`.
  * Support subtree-level logical properties, where setting the `dir` prop to `ltr` or `rtl` alters the way logical properties are resolved. Currently, React Native only supports global writing direction configuration for Android.
* CSS Animations
  * Support declarative keyframes and animations that can be optimized on the native side and avoid the need for `Animated`.
  * See [animation](https://developer.mozilla.org/en-US/docs/Web/CSS/animation)).
* CSS Transitions
  * Support declarative transitions that can be optimized on the native side and avoid the need for `Animated`.
  * Support `transitionProperty`, `transitionDuration`, `transitionTimingFunction`, and `transitionDelay` (see [transition](https://developer.mozilla.org/en-US/docs/Web/CSS/transition)).
* `aspectRatio`
  * Support string values, i.e., `'16 / 9'`, to align with CSS.
* `backgroundImage`
  * Support setting background images via `url()`.
  * Optional: support CSS gradients.
* `boxShadow`
  * Add native support for CSS box shadows.
  * This would replace the very buggy iOS-specific `shadow*` styles and Android `elevation` style, allowing developers to unify shadows across native and web platforms.
* `filter`
 * Support declarative filters per [filter](https://developer.mozilla.org/en-US/docs/Web/CSS/filter)
* `fontVariant`
  * Support space-separated string values to align with CSS.
* `fontWeight`
  * Support number values to align with React DOM / CSS.
* `lineClamp`
  * Support CSS clamping text to a set number of lines via style.
  * This would be equivalent to using the `numberOfLines` prop on `<Text>` components.
* `objectFit`
  * Support CSS `objectFit` property for `<Image>`.
  * This would be a subset (no "repeat" keyword) of functionality currently supported by the `resizeMode` prop and style for `<Image>`.
  * `objectFit === 'contain'` => `resizeMode = 'contain'`
  * `objectFit === 'cover'` => `resizeMode = 'cover'`
  * `objectFit === 'fill'` => `resizeMode = 'stretch'`
  * `objectFit === 'scale-down'` => `resizeMode = 'contain'`
* `pointerEvents`
  * Support setting the `pointerEvents` values via style rather than a prop.
  * Retain the React Native specific `box-none` and `box-only` values.
* `textShadow`
  * Add native support for CSS text shadows.
* `transform`
  * Support using string values to set transforms.
  * This can be shimmed in user-space with a parser.
* `verticalAlign`
  * Make `verticalAlign` the same as `textAlignVertical`.
  * `verticalAlign === 'middle'` is the same as `textAlignVertical = 'center'`;
* `userSelect`
  * Support setting ability to select text via style.
  * Equivalent to using `selectable` prop on `<Text>`.

```jsx
<View
  style={{
    boxShadow: '1px 1px 1px 1px #eee',
    backgroundColor: 'hsla(50,50,50,0.5)',
    pointerEvents: 'none',
    transform: 'scale(0.9)',
    width: '5rem'
  }}
>

<Text
  style={{
    lineClamp: 3,
    userSelect: 'none',
    verticalAlign: 'middle'
  }}
>

<Image
  style={{
    objectFit: 'cover'
  }}
>
```

### New imperative APIs

* `EventTarget` API
  * The `EventTarget` API is implemented in JS environments, and should be available on React Native host elements. This feature is commonly used on web, and could be used by React Native developers to support more complex features without first requiring further core API changes to support each use case.
  * Promote `node.unstable_addEventListener`, `node.unstable_removeEventListener` to stable APIs.
  * Add `node.dispatchEvent` support.
  * This cannot be shimmed in user-space.
* `node.getBoundingClientRect`
  * Synchronous element measurement API. Current APIs like `node.measure` are async and return slightly different information to what you expect from `getBoundingClientRect` on web.
  * This cannot be shimmed in user-space.

### Other

* Revisit the type of all events modeled on W3C events. For example, `onTouchStart` should receive an event object that conforms to that received by the same prop in React DOM.
* Aim to deprecate `StyleSheet.flatten()`.
  * This API encourages runtime introspection of styles in render calls. This pattern is a performance cost (flattening arrays of objects), and prevents us from supporting build-time optimizations for web (extracting styles to CSS files).
* Aim to deprecate `StyleSheet.compose()`.
  * This API has no real value.
* Aim to restore obfuscation of styles passed to `StyleSheet.create()`.
  * Returning the styles from create calls encourages introspection of those styles. On web, we need to be able to remove the JavaScript style objects from bundles to support build-time optimization like extraction to CSS files.
* Aim to deprecate the `Touchable*` components.
  * The `Pressable` component was shipped as a replacement for `Touchable*`.
* Aim to support more interaction states with `Pressable`.
  * Add ‘hovered’ and ‘focused’ to the state object, and replace ‘testOnly_pressed’ with an object of states.
* Aim to move `Animated` out of core.
  * Move Animated out of core to separate package in monorepo, allow React Native for Web to import this package. Eventually could drop Animated from ‘react-native’ export.
* Aim to move `VirtualizedList` and `FlatList` out of core.
  * Move `VirtualizedList`/`FlatList` out of core to separate package in monorepo, allow React Native for Web to import this package. Eventually could drop these components from `'react-native'` export.
* Remove Flow types and JSX before publishing to npm (make this the default for the community). React Native and its community packages currently publish uncompiled, flow-typed JavaScript to npm. This causes common tools to crash on these packages.
* Prevent libraries importing `'react-native'` package internals, by using the Node.js package exports API.

## Drawbacks

* Many of these features can be shimmed in user-space, at the cost of runtime overhead for native or web platforms. The runtime overhead for web is untenable. The runtime overhead of shifting shims into the JS targeting native has yet to be established.
* It adds redundancy to the React Native API, as the proposal is to make additive changes and avoid requiring removal of existing APIs in the short term.

## Adoption strategy

To encourage library and product developers to proactively migrate to these new APIs, React Native for Web plans to only support these W3C standards-based APIs in future versions. This will allow us to incrementally add APIs to React Native without needing to commit to simultaneously deprecating APIs and migrating existing React Native code. Existing React Native developers will adopt these APIs if they wish to support web.

There are few, if any, breaking changes anticipated for React Native itself.

## How we teach this

Teaching these APIs is simplified by developer familiarity with existing DOM / React DOM APIs.
