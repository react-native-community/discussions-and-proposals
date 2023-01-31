---
title: React DOM for Native
author:
- Nicolas Gallagher
date: 2022-08-03
updated: 2023-01-31
---

# RFC: React DOM for Native

## Summary

This is a proposal to incrementally reduce the API fragmentation faced by developers using React to target multiple platforms via code shared between native and web. The proposed cross-platform user interface APIs are a subset of existing web standards for DOM, CSS, and HTML - "Strict DOM". The proposed changes are overwhelmingly additive and do not require migration of existing React Native UI code (deprecations are optional / follow up work). Incremental progress can help to reduce the fragmentation between React Native and React DOM components, while the ability to run React DOM code (with minor modifications) on native is a longer-term goal.

## Motivation

React Native currently includes many APIs that are modelled on Web APIs but do not conform to the standards of those Web APIs. React Native also includes many APIs that achieve the same results on Android and iOS but are exposed as 2 different props. And React Native includes several APIs that have known performance (network and runtime) drawbacks on Web.

This proposal aims to allow developers to target more platforms with cross-platform APIs, and deliver better performance when targeting browsers. Features for Android, iOS, and Web are unified by aligning with the Web standard. Supporting standards helps to:

* minimize the overhead when running in browsers;
* reduce developer education required to learn features;
* set clear and cohesive API end-state expectations for contributors to aim for;
* accelerate the framework's development by avoiding API design costs;
* support backwards compatibility;
* develop universal codebases.

The existing solution for targeting web with React Native is to use React Native for Web. React Native for Web is a user-space library built on top of React DOM and native DOM APIs. It shims React Native components and APIs on the web. The tooling for an existing React DOM app simply maps the 'react-native' export to 'react-native-web'.

![](https://user-images.githubusercontent.com/239676/205388174-6fce19f1-8b3a-4c6e-82f1-89ef9c21ab7d.png)

This is the most complete and widely used shim, but comes with considerable DX and UX costs on the web. The shim must implement a large surface area of fragmented APIs, and it needs to modify standard APIs and objects (e.g., events) to match React Native's non-standard implementations.

In contrast, implementing a "Strict DOM" subset in React Native shifts the weight of bridging native and web apps onto React Native, where it can be done most efficiently. Although React Native will not support all the features available on web, it will still support a greater expanded feature set relative to React Native today. On the web, we would only need to combine React DOM with a white-label CSS compiler like stylex.

![](https://user-images.githubusercontent.com/239676/205388313-e58f8793-51cd-4a6e-86e7-89425fd2d0e1.png)

## Detailed design

The "complete" API involved is broken down into the following sections.

* [Environment API](#environment-api). Global APIs to add to the host environment.
* [Elements API](#elements-api-dom-subset). DOM APIs to add to host component instances.
* [Components API](#components-api-dom-subset). React DOM components to add to React Native.
* [Props API](#props-api-react-dom-subset). React DOM component props to add to new components, and existing React Native components as part of an incremental strategy.
* [Styling API](#styles-api-css-subset). CSS APIs to add to the React Native environment.


A simplified example of the user-space code we'd aim to support is as follows:

```js
import { html, css } from 'react-native/dom';

export function VStackPanel(props) {
  const connectedCallback = function (node) {
    const handler = () => {
      const { offsetWidth } = e.target;
      // ...
    };
    const options = { capture: true };
    node.addEventListener('pointerdown', handler, options);
    return function disconnected() {
      node.removeEventListener('pointerdown', handler, options)
    }
  };

  return (
    // Use HTML element
    <html.div
      children={props.children}
      // Synchronous click event that matches W3C event type
      on-click={props.click}
      // Use HTMLElement instance API
      ref={connectedCallback}
      // Use optimized styles
      style={css(
        { flexDirection: 'column', flex: 1 },
        props.style
      )}
    />
  )
}
```

But there are many incremental steps that can be taken along the way.

## Implementation and adoption strategy

The proposal is to support both an incremental adoption strategy, as well as a separate "React DOM" bindings exports. The incremental strategy would find ways to align existing APIs in React Native with their React DOM equivalents, and gradually deprecate the non-standard APIs. Whereas the separate bindings export would be designed to provide most of what is needed for a direct compatibility layer with React DOM.

This would ensure a significant amount of overlap between the existing React Native components and the dedicated "React DOM" compatibility API, allowing for co-mingling of code and reduced developer education. There is no suggestion of deprecating the current React Native components and APIs at this stage.

### Reduce API fragmentation across platforms

The initial step is to incrementally reduce existing API fragmenetation across platforms. This involves consolidating the different props, styles, and events for similar functionality across different platforms. The APIs that we consolidate upon are the W3C standards, as React Native is already fairly closely aligned with these standards.

Doing this will make it easier for web developers to work with React Native's existing components and related APIs. This is the path for developers to gradually migrate their existing React Native code into a form that is more aligned with the web. And it will allow React Native for Web to be smaller and faster, providing better results on web via the existing tools that developers use today.

One example of this kind of incremental change is to map props in the existing core components (i.e., `View`, `Text`, etc.) to the W3C equivalents: `id` to `nativeID`, `aria-label` to `accessibilityLabel`, and so on. React Native for Web currently "owns" this translation step, but we would instead move it into React Native to claw back performance on web. For developers familiar with React DOM, there will be less of a learning curve to applying that knowledge to React Native.

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

```jsx
<Image
  alt="Alternative text"
  crossOrigin="anonymous"
  srcSet="https://image.png 1x, https://image2.png 2x"
  width={320}
>
```

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

Example of how to map `srcSet` to `source`:

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

Example of how to map `autoComplete` to Android's `autoComplete` and iOS's `textContentType`:

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

Where possible we should make these changes to the native code. However, we will prioritize user-space shims in JavaScript where they are faster to ship and allow us to gather feedback/support for this overall direction.

Beyond props, there is also the opportunity to reduce fragementation in styling APIs.

* [ ] Don't hard crash on unknown or invalid style properties and values.
* [ ] Deprecate `StyleSheet.absoluteFill`. Use `position:'absolute';inset:0;` instead.
* [ ] Deprecate `StyleSheet.absoluteFillObject`.
* [ ] Deprecate `StyleSheet.compose()`. Use existing `[ a, b ]` syntax instead.
* [ ] Deprecate `StyleSheet.flatten()`. This API encourages runtime introspection of styles in render calls. This pattern is a performance cost (flattening arrays of objects), and prevents us from supporting build-time optimizations for web (e.g., extracting styles to CSS files).
* [ ] Deprecate or rename `setStyleAttributePreprocessor` (e.g., `unstable_setStyleAttributePreprocessor`).
* [ ] `StyleSheet.create()` should obfuscate styles to prevent introspection (i.e., revert identify function change). On web, we need to be able to remove the JavaScript style objects from bundles to support build-time optimization like extraction to CSS files.

To encourage React Native library and product developers to proactively migrate to these new APIs, React Native for Web plans to only support these W3C standards-based APIs in future versions. This will allow us to incrementally add APIs to React Native without needing to commit to simultaneously deprecating APIs and migrating existing React Native code. Existing React Native developers must adopt these APIs if they wish to support web.

### Crate a separate package or export for DOM bindings

The next step would be to begin exporting separate React DOM bindings from React Native. This would aim to allow code originally written with React DOM to run on React Native with only minor modifications. There would be no need to use React Native for Web, and the web layer in this scenario would be a lot smaller, limited mostly to implementing the modern W3C events API and integrating a style API that allows for static extraction to optimized CSS.

Even this can be done in incremental fashion, with early (user-space) prototypes to get feedback. Most tags can be mapped to existing React Native components and props, in the reverse of the [mapping used by React Native for Web](https://github.com/necolas/react-native-web/blob/b2eb3ca03ffc4b8570591b6c2082c6e4edae0e7f/packages/react-native-web/src/modules/AccessibilityUtil/propsToAccessibilityComponent.js#L12-L30).

### Equivalent API for Web

In parallel, a user-space layer over React DOM would ensure equivalent APIs between native and web, by converging them both towards a desired the "end state" where the differences between React Native and React DOM are erased while landing on a "modern" flavor of React DOM.

The benefit of initially doing this in user-space is that it reduces the burden of complex migratations for React Native and React DOM (e.g., the changes to React DOM event types, the `style` prop, and explicit imports of component types), while allowing us to validate and iterate on the proposed end-state APIs with interested parties. This is currently blocked on OSS-ing of stylex.

## How we teach this

Teaching these APIs is simplified by developer familiarity with existing DOM / React DOM APIs.

## Drawbacks

Many of these features can be shimmed in user-space, at the cost of runtime overhead for native or web platforms. The runtime overhead for web is untenable. The runtime overhead of shifting shims into the React Native JS has yet to be established.

This adds redundancy to the React Native API, as the proposal is to make additive changes and avoid requiring removal of existing APIs in the short term.

We have to be very intentional about how much compatibility with open web standards we can promise. Some cross-platform limitations are rooted in significant differences in the host platforms APIs (e.g., accessibility) and the lack of 99% compat may frustrate users.

Aiming for a degree of React DOM compatibility shifts the burden of missing features onto React Native. Developers may not be used to an API that is complete on web but partially implemented on native platforms.

-----

# Environment APIs

## `window`

### Events

* [ ] Receives all W3C events dispatched to components (capture and bubble phase). Resolves proposal [#249](https://github.com/react-native-community/discussions-and-proposals/issues/249).
* [ ] Dispatches [`resize`](https://developer.mozilla.org/en-US/docs/Web/API/Window/resize_event).

### Properties

* [ ] [`devicePixelRatio`](https://developer.mozilla.org/en-US/docs/Web/API/Window/devicePixelRatio)
* [ ] [`navigator`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator)
  * [ ] [`clipboard`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/clipboard)
  * [ ] [`languages`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/languages)
  * [ ] [`permissions`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/permissions)
  * [ ] [`vibrate()`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/vibrate)
* [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/performance_property)

### Methods

* [ ] `addEventListener()`
* [ ] `dispatchEvent()`
* [ ] `removeEventListener()`

### APIs

* [ ] [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch)
* [ ] [`matchMedia`](https://developer.mozilla.org/en-US/docs/Web/API/Window/matchMedia). Resolves proposal [#350](https://github.com/react-native-community/discussions-and-proposals/issues/350).
* [ ] [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver) API for observing the intersection of target elements. This can also be used as a building block for performance tooling.
* [ ] [`MutationObserver`](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) API for watching changes to the host node tree. This can also be used as a building block for performance tooling.
* [ ] [`ResizeObserver`](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver) API for responding the changes in the size of elements. Note that this API no longer includes positional coordinates and is optimized for width/height information.

## `document`

Support the `document` object for common patterns such as listening to visibility changes, and listening to capture or bubble phase events for the entire application (e.g., "outside click" pattern), etc.

### Events

* [ ] [`scroll`](https://developer.mozilla.org/en-US/docs/Web/API/Document/scroll_event).
* [ ] [`visibilitychange`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event). Can replace the `AppState` API.

### Properties

- [ ] [`activeElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/activeElement).
- [ ] [`visibilityState`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilityState)

### Methods

- [ ] [`getElementFromPoint(x,y)`](https://developer.mozilla.org/en-US/docs/Web/API/Document/elementFromPoint). Resolves proposal [#501](https://github.com/react-native-community/discussions-and-proposals/issues/501).

-----

# Elements API (DOM subset)

Subset of cross-platform DOM APIs exposed on element instances. This would be an imperative API for working with the Shadow tree in React Native. Based on data from [MDN Web API](https://developer.mozilla.org/en-US/docs/Web/API), please refer to the MDN links for complete DOM APIs and comment if you believe any should be included in the proposed subset.

In practice, this DOM API subset is generally limited to the “best practice” APIs that can safely be used with React DOM (i.e., read-only operations that don’t modify the DOM tree), and most React components should already be limited to this subset. One of the important details of this API are how it supports the different ways of [determining the dimensions of elements](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Object_Model/Determining_the_dimensions_of_elements).

## EventTarget

The `EventTarget` API is implemented in JS environments, and should be available on React Native host elements. This feature is commonly used on web, and could be used by React Native developers to support more complex features without first requiring further core API changes to support each use case.

### Methods

- [ ] [`EventTarget.addEventListener()`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener). Registers an event handler to a specific event type on the element.
- [ ] [`EventTarget.dispatchEvent()`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent). Dispatches an event to this node in the DOM and returns a boolean value that indicates whether no handler canceled the event.
- [ ] [`EventTarget.removeEventListener()`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/removeEventListener). Removes an event listener from the element.

## Event and CustomEvent

[Event()](https://developer.mozilla.org/en-US/docs/Web/API/Event/Event) and [CustomEvent()](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent) represent events initialized by developers and that can be dispatched to elements using `EventTarget.dispatchEvent()`.

## Node

The DOM Node interface is an abstract base class upon which many other DOM API objects are based, thus letting those object types to be used similarly and often interchangeably. All objects that implement Node functionality are based on one of its subclasses. Every kind of host node is represented by an interface based on Node.

In some cases, a particular feature of the base Node interface may not apply to one of its child interfaces; in that case, the inheriting node may return null or throw an exception, depending on circumstances.

### Methods

Node inherits methods from its parent, EventTarget.

- [ ] [`Node.compareDocumentPosition()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/compareDocumentPosition)
- [ ] [`Node.contains()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/contains). Returns true or false value indicating whether or not a node is a descendant of the calling node.
- [ ] [`Node.getRootNode()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode). Returns the context object's root which optionally includes the shadow root if it is available..

## Element

Element is the most general base class from which all element objects (i.e. objects that represent elements) in a Document inherit. It only has methods and properties common to all kinds of elements. More specific classes inherit from Element.

### Properties

Element inherits properties from its parent interface, Node, and by extension that interface's parent, EventTarget.

- [ ] [`Element.clientHeight`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientHeight) [Read only]. Returns a number representing the inner height of the element.
- [ ] [`Element.clientLeft`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientLeft) [Read only]. Returns a number representing the width of the left border of the element.
- [ ] [`Element.clientTop`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientTop) [Read only]. Returns a number representing the width of the top border of the element.
- [ ] [`Element.clientWidth`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientWidth) [Read only]. Returns a number representing the inner width of the element.
- [ ] [`Element.id`](https://developer.mozilla.org/en-US/docs/Web/API/Element/id) [Read only]. Is a DOMString representing the id of the element.
- [ ] [`Element.scrollHeight`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollHeight) [Read only]. Returns a number representing the scroll view height of an element.
- [ ] [`Element.scrollLeft`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollLeft). Is a number representing the left scroll offset of the element.
- [ ] [`Element.scrollLeftMax`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollLeftMax) [Read only]. Returns a number representing the maximum left scroll offset possible for the element.
- [ ] [`Element.scrollTop`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop). A number representing number of pixels the top of the element is scrolled vertically.
- [ ] [`Element.scrollTopMax`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTopMax) [Read only]. Returns a number representing the maximum top scroll offset possible for the element.
- [ ] [`Element.scrollWidth`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollWidth) [Read only]. Returns a number representing the scroll view width of the element.

### Methods

Element inherits methods from its parents Node, and its own parent, EventTarget.

- [ ] [`Element.computedStyleMap()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/computedStyleMap). Returns a StylePropertyMapReadOnly interface which provides a read-only representation of a CSS declaration block that is an alternative to CSSStyleDeclaration.
- [ ] [`Element.getAttribute()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAttribute). Returns the value of a specified attribute on the element.
- [ ] [`Element.getBoundingClientRect()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect). Returns the size of an element and its position relative to the viewport.
- [ ] [`Element.getClientRects()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getClientRects). Returns a collection of `DOMRect` objects that indicate the bounding rectangles for each CSS border box in a client.
- [ ] [`Element.hasAttribute()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/hasAttribute). Returns a Boolean value indicating whether the specified element has the specified attribute or not.
- [ ] [`Element.hasPointerCapture()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/hasPointerCapture). Checks whether the element on which it is invoked has pointer capture for the pointer identified by the given pointer ID.
- [ ] [`Element.scroll()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scroll). Scrolls to a particular set of coordinates inside a given element.
- [ ] [`Element.scrollBy()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollBy). Scrolls an element by the given amount.
- [ ] [`Element.scrollIntoView()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollIntoView). Scrolls the page until the element gets into the view.
- [ ] [`Element.scrollTo()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTo). Scrolls to a particular set of coordinates inside a given element.
- [ ] [`Element.setPointerCapture()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/setPointerCapture). Used to designate a specific element as the capture target of future pointer events. Subsequent events for the pointer will be targeted at the capture element until capture is released.
- [ ] [`Element.releasePointerCapture()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/releasePointerCapture). Releases (stops) pointer capture that was previously set for a specific (PointerEvent) pointer.

### Events

Listen to these events using `addEventListener()` or by assigning an event handler to the equivalent component prop.

- [ ] `error`. Fired when a resource failed to load, or can't be used. For example, if a script has an execution error or an image can't be found or is invalid. Also available via the `on-error` property.
- [ ] `scroll`. Fired when the document view or an element has been scrolled. Also available via the `on-scroll` property.
- [ ] `select`. Fired when some text has been selected. Also available via the `on-select` property.
- [ ] `wheel`. Fired when the user rotates a wheel button on a pointing device (typically a mouse). Also available via the `on-wheel` property.

Clipboard events

- [ ] [`copy`](https://developer.mozilla.org/en-US/docs/Web/API/Element/copy_event). Fired when the user initiates a copy action through the browser's user interface. Also available via the `on-copy` property.
- [ ] [`cut`](https://developer.mozilla.org/en-US/docs/Web/API/Element/cut_event). Fired when the user initiates a cut action through the browser's user interface. Also available via the `on-cut` prop.
- [ ] [`paste`](https://developer.mozilla.org/en-US/docs/Web/API/Element/paste_event). Fired when the user initiates a paste action through the browser's user interface. Also available via the `on-paste` prop.

Focus events

- [ ] [`blur`](https://developer.mozilla.org/en-US/docs/Web/API/Element/blur_event). Fired when an element has lost focus. Also available via the `on-blur` prop.
- [ ] [`focus`](https://developer.mozilla.org/en-US/docs/Web/API/Element/focus_event). Fired when an element has gained focus. Also available via the `on-focus` prop.
- [ ] [`focusin`](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusin_event). Fired when an element is about to gain focus.
- [ ] [`focusout`](https://developer.mozilla.org/en-US/docs/Web/API/Element/focusout_event). Fired when an element is about to lose focus.

Keyboard events

- [ ] [`keydown`](https://developer.mozilla.org/en-US/docs/Web/API/Element/keydown_event). Fired when a key is pressed. Also available via the `on-keydown` prop.
- [ ] [`keyup`](https://developer.mozilla.org/en-US/docs/Web/API/Element/keyup_event). Fired when a key is released. Also available via the `on-keyup` prop.

Pointer events

- [ ] [`auxclick`](https://www.w3.org/TR/pointerevents3/#the-click-auxclick-and-contextmenu-events). Fired when a non-primary pointing device button (e.g., any mouse button other than the left button) has been pressed and released on an element. Also available via the `onAuxClick` prop.
- [ ] [`click`](https://www.w3.org/TR/pointerevents3/#the-click-auxclick-and-contextmenu-events). Fired when a pointing device button (e.g., a mouse's primary button) is pressed and released on a single element. Also available via the `onClick` prop.
- [ ] [`contextmenu`](https://www.w3.org/TR/pointerevents3/#the-click-auxclick-and-contextmenu-events). Fired when the user attempts to open a context menu. Also available via the oncontextmenu property.
- [ ] [`gotpointercapture`](https://www.w3.org/TR/pointerevents3/#the-gotpointercapture-event). Fired when an element captures a pointer using `setPointerCapture()`. Also available via the `on-gotpointercapture` prop.
- [ ] [`lostpointercapture`](https://www.w3.org/TR/pointerevents3/#the-lostpointercapture-event). Fired when a captured pointer is released. Also available via the `on-lostpointercapture` prop.
- [ ] [`pointercancel`](https://www.w3.org/TR/pointerevents3/#the-pointercancel-event). Fired when a pointer event is canceled. Also available via the `on-pointercancel` prop.
- [ ] [`pointerdown`](https://www.w3.org/TR/pointerevents3/#the-pointerdown-event). Fired when a pointer becomes active. Also available via the `on-pointerdown` prop.
- [ ] [`pointerenter`](https://www.w3.org/TR/pointerevents3/#the-pointerenter-event). Fired when a pointer is moved into the hit test boundaries of an element or one of its descendants. Also available via the `on-pointerenter` prop.
- [ ] [`pointerleave`](https://www.w3.org/TR/pointerevents3/#the-pointerleave-event). Fired when a pointer is moved out of the hit test boundaries of an element. Also available via the `on-pointerleave` prop.
- [ ] [`pointermove`](https://www.w3.org/TR/pointerevents3/#the-pointermove-event). Fired when a pointer changes coordinates. Also available via the `on-pointermove` prop.
- [ ] [`pointerout`](https://www.w3.org/TR/pointerevents3/#the-pointerout-event). Fired when a pointer is moved out of the hit test boundaries of an element (among other reasons). Also available via the `on-pointerout` prop.
- [ ] [`pointerover`](https://www.w3.org/TR/pointerevents3/#the-pointerover-event). Fired when a pointer is moved into an element's hit test boundaries. Also available via the `on-pointerover` prop.
- [ ] [`pointerrawupdate`](https://www.w3.org/TR/pointerevents3/#the-pointerrawupdate-event). Fired when a pointer changes any properties that don't fire `pointerdown` or `pointerup` events.
- [ ] [`pointerup`](https://www.w3.org/TR/pointerevents3/#the-pointerup-event). Fired when a pointer is no longer active. Also available via the `on-pointerup` prop.

## HTMLElement

The HTMLElement interface represents any HTML element. Some elements directly implement this interface, while others implement it via an interface that inherits it.

### Properties

Inherits properties from its parent, Element, and implements those from DocumentAndElementEventHandlers, GlobalEventHandlers, and TouchEventHandlers.

- [ ] [`HTMLElement.hidden`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/hidden) [Read only]. A boolean value indicating if the element is hidden or not.
- [ ] [`HTMLElement.offsetHeight`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetHeight) [Read only]. Returns a double containing the height of an element, relative to the layout.
- [ ] [`HTMLElement.offsetLeft`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetLeft) [Read only]. Returns a double, the distance from this element's left border to its offsetParent's left border.
- [ ] [`HTMLElement.offsetParent`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetParent) [Read only]. Returns a Element that is the element from which all offset calculations are currently computed.
- [ ] [`HTMLElement.offsetTop`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetTop) [Read only]. Returns a double, the distance from this element's top border to its offsetParent's top border.
- [ ] [`HTMLElement.offsetWidth`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetWidth) [Read only]. Returns a double containing the width of an element, relative to the layout.

### Methods

Inherits methods from its parent, Element, and implements those from DocumentAndElementEventHandlers, GlobalEventHandlers, and TouchEventHandlers.

- [x] [`HTMLElement.blur()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/blur). Removes keyboard focus from the currently focused element.
- [ ] [`HTMLElement.click()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/click). Sends a mouse click event to the element.
- [ ] [`HTMLElement.focus(options)`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus). Makes the element the current keyboard focus.

### Events

Listen to these events using `addEventListener()` or by assigning an event listener to the equivalent component prop.

- [ ] [`beforeinput`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/beforeinput_event). Fired when the value of an `<input>`, `<select>`, or `<textarea>` element is about to be modified.
- [ ] [`change`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/change_event). Fired when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed and committed by the user. Unlike the input event, the change event is not necessarily fired for each alteration to an element's value. Also available via the `on-change` property.
- [ ] [`input`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/input_event). Fired when the value of an `<input>`, `<select>`, or `<textarea>` element has been changed. Also available via the `on-input` property.

## HTMLImageElement

The HTMLImageElement interface represents an HTML `<img>` element, providing the properties and methods used to manipulate image elements.

### Properties

Inherits properties from its parent, HTMLElement.

- [ ] [`HTMLImageElement.complete`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/complete) [Read only]. Returns a boolean value that is true if the browser has finished fetching the image, whether successful or not. That means this value is also true if the image has no src value indicating an image to load.
- [ ] [`HTMLImageElement.currentSrc`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/currentSrc) [Read only]. Returns a USVString representing the URL from which the currently displayed image was loaded. This may change as the image is adjusted due to changing conditions, as directed by any media queries which are in place.
- [ ] [`HTMLImageElement.naturalHeight`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/naturalHeight) [Read only]. Returns an integer value representing the intrinsic height of the image in CSS pixels, if it is available; else, it shows 0. This is the height the image would be if it were rendered at its natural full size.
- [ ] [`HTMLImageElement.naturalWidth`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/naturalWidth) [Read only]. An integer value representing the intrinsic width of the image in CSS pixels, if it is available; otherwise, it will show 0. This is the width the image would be if it were rendered at its natural full size.

### Events

- [ ] `error`. Also available via the `on-error` prop.
- [ ] `load`. Also available via the `on-load` prop.

## HTMLInputElement

The HTMLInputElement interface provides special properties and methods for manipulating the options, layout, and presentation of `<input>` elements.

### Properties

Properties that apply only to visible elements containing text or numbers

- [ ] [`disabled`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/disabled).
- [ ] [`selectionEnd`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement#selectionend). unsigned long: Returns / Sets the end index of the selected text. When there's no selection, this returns the offset of the character immediately following the current text input cursor position.
- [ ] [`selectionStart`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement#selectionstart). unsigned long: Returns / Sets the beginning index of the selected text. When nothing is selected, this returns the position of the text input cursor (caret) inside of the <input> element. (Resolves issue (#35616)[https://github.com/facebook/react-native/pull/35616].)
- [ ] [`selectionDirection`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement#selectiondirection). string: Returns / Sets the direction in which selection occurred. Possible values are: forward (the selection was performed in the start-to-end direction of the current locale), backward (the opposite direction) or none (the direction is unknown).
- [ ] [`value`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement#value). string: Returns / Sets the current value of the control. If the user enters a value different from the value expected, this may return an empty string.

### Methods

- [ ] [`select()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/select). Selects all the text in the input element, and focuses it so the user can subsequently replace all of its content.
- [ ] [`setSelectionRange()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/setSelectionRange). Selects a range of text in the input element (but does not focus it).
- [ ] [`showPicker()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/showPicker). Shows a browser picker for date, time, color, and files.

### Events

Listen to these events using `addEventListener()` or by assigning an event listener to the oneventname property of this interface:

- [ ] [`invalid`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/invalid_event). Fired when an element does not satisfy its constraints during constraint validation. Also available via the `on-invalid` prop.
- [ ] [`select`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/select_event) event. Fired when some text has been selected.
- [ ] [`selectionchange`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/selectionchange_event) event. Fires when the text selection in a `<input>` element has been changed. Also available via the `on-selectionchange` prop.

## HTMLTextAreaElement

The HTMLInputElement interface provides special properties and methods for manipulating the options, layout, and presentation of `<input>` elements.

### Properties

- [ ] [`disabled`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/disabled).
- [ ] `selectionEnd`. unsigned long: Returns / Sets the end index of the selected text. When there's no selection, this returns the offset of the character immediately following the current text input cursor position.
- [ ] `selectionStart`. unsigned long: Returns / Sets the beginning index of the selected text. When nothing is selected, this returns the position of the text input cursor (caret) inside of the `<textarea>` element.
- [ ] `selectionDirection`. string: Returns / Sets the direction in which selection occurred. Possible values are: forward (the selection was performed in the start-to-end direction of the current locale), backward (the opposite direction) or none (the direction is unknown).
- [ ] `value`. string: Returns / Sets the current value of the control. If the user enters a value different from the value expected, this may return an empty string.

### Methods

- [ ] `select()`. Selects all the text in the input element, and focuses it so the user can subsequently replace all of its content.
- [ ] `setSelectionRange()`. Selects a range of text in the input element (but does not focus it).

### Events

Listen to these events using `addEventListener()` or using the equivalent prop name.

- [ ] [`select`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/select_event) event. Fired when some text has been selected.
- [ ] [`selectionchange`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/selectionchange_event) event. Fires when the text selection in a `<input>` element has been changed. Also available via the `on-selectionchange` prop.

-----

# Components API (React DOM subset)

Support a subset of React DOM components directly. Exported as an `html` object, i.e., `html.div`, `html.a`, etc.

This would provide a built-in declarative API for cross-platform elements in React Native. All unknown tags default to `<span>` layout. All known-but-unimplemented block-level tags default to `<div>` layout.

## Sections

Content sectioning elements allow you to organize the document content into logical pieces. Use the sectioning elements to create a broad outline for your page content, including header and footer navigation, and heading elements to identify sections of content.

* [ ] [`article`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/article) is equivalent to `View` with `display:block` layout and `role="article"`.
* [ ] [`aside`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/aside).
* [ ] [`div`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/div) is equivalent to `View` with `display:block` layout.
* [ ] [`footer`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/footer).
* [ ] [`h1-6`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/Heading_Elements).
* [ ] [`header`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/header).
* [ ] [`main`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/main).
* [ ] [`nav`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/nav).
* [ ] [`section`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/section) is equivalent to `View` with `display:block` layout and `role="section"`.

## Text

Text content elements organize blocks or sections of content. Important for accessibility, these elements identify the purpose or structure of that content.

* [ ] [`blockquote`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/blockquote).
* [ ] [`li`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/li).
* [ ] [`ol`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/ol).
* [ ] [`ul`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/ul).
* [ ] [`p`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/p).
* [ ] [`pre`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/pre).

## Inline text

Inline text semantics define the meaning, structure, or style of a word, line, or any arbitrary piece of text.

* [ ] [`a`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a).
* [ ] [`b`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/b).
* [ ] [`bdi`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/bdi).
* [ ] [`bdo`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/bdo).
* [ ] [`code`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/code).
* [ ] [`em`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/em).
* [ ] [`span`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/span) is equivalent to `Text` with `display:inline` layout.
* [ ] [`strong`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/strong).
* [ ] [`sub`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/sub).
* [ ] [`sup`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/sup).

## Media

* [ ] [`img`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img) is similar to `Image` without advanced loader configuration.

## Forms

Elements which can be used together to create forms which the user can fill out and submit. There's a great deal of further information about this available in the [HTML forms guide](https://developer.mozilla.org/en-US/docs/Learn/Forms).

* [ ] [`button`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/button).
* [ ] [`input`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input) is equivalent to `TextInput` with `inputMode` corresponding to `type`. Expanding input type support to number, datetime, color, etc., resolves proposal [#510](https://github.com/react-native-community/discussions-and-proposals/issues/510).
* [ ] [`label`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/label).
* [ ] [`optgroup`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/optgroup).
* [ ] [`option`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/option).
* [ ] [`progress`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/progress).
* [ ] [`select`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/select).
* [ ] [`textarea`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/textarea) is equivalent to `TextInput` with `multiline={true}`.

-----

# Props API (React DOM subset)

Support a subset of React DOM props.

## Common props

Props supported on all elements.

Supporting the ARIA 1.2 interface would allow more ARIA-targeting React DOM libraries to work on React Native, and would provide a well defined cross-platform target for equivalent native features (e.g., React Native for Windows needing heading levels, set size, etc.)

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
- [ ] ([`data-*`]() props. Resolves proposal [#323](https://github.com/react-native-community/discussions-and-proposals/issues/323))
- [ ] [`dir`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/dir) configures element's layout / writing direction.
- [ ] [`elementTiming`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/elementtiming).
- [ ] [`lang`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/lang) identifies the language of the text content.
- [ ] [`hidden`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/hidden) is the same as `display:'none'`.
- [x] [`id`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/id) is the same as `nativeID`.
- [ ] [`inert`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inert).
- [x] [`role`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles) is the same as `accessibilityRole` (with support for web values).
- [x] `style` is unchanged from existing React Native.
- [x] [`tabIndex`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/tabindex) is the same as `focusable` (Android, Desktop). Add support for `0` and `-1` values only.

## Common events

Events supported on all elements.

Each event handler should receive a W3C event object of the corresponding type, not a synthetic event. These props are merely conveniences and otherwise identical to using the `EventTarget` API mentioned later on. Note that no capture phase props are included, as listening to the capture phase (as well as custom events) can be handled with the `EventTarget` API. (Alternatively, we introduce `oncapture-click`, etc.)

To avoid breaking changes in React Native and React DOM, it may be preferrable to create new event prop names that receive events as specified by web standards, e.g., `on-click`, `on-pointerup`.

Resolves proposal [#492](https://github.com/react-native-community/discussions-and-proposals/issues/492).

- [ ] `on-auxclick` is a `PointerEvent` for auxillary clicks.
- [ ] `on-blur`.
- [ ] `on-click` is a `PointerEvent` for primary clicks.
- [ ] `on-contextmenu` is a `PointerEvent` for context clicks / long press.
- [ ] `on-copy`.
- [ ] `on-cut`.
- [ ] `on-focus`.
- [ ] `on-gotpointercapture`.
- [ ] `on-lostpointercapture`.
- [ ] `on-paste`.
- [ ] `on-pointercancel`.
- [ ] `on-pointerdown`.
- [ ] `on-pointerenter`.
- [ ] `on-pointerleave`.
- [ ] `on-pointermove`.
- [ ] `on-pointerout`.
- [ ] `on-pointerover`.
- [ ] `on-pointerup`.
- [ ] `on-keydown`.
- [ ] `on-keyup`.

## `<a>` props

- [ ] [`download`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-download).
- [ ] [`href`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-href). Clicking will always attempt to open the URL in a browser, mail client, etc. Use of `e.preventDefault()` in an event handler will disable this behavior and allow custom routing logic to handle the interaction.
- [ ] [`referredPolicy`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-referrerpolicy).
- [ ] [`rel`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-rel).
- [ ] [`target`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#attr-target).

## `<img>` props

Additional props for `<img>`.

- [x] [`alt`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-alt) prop for alternative text support.
- [x] [`crossOrigin`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-crossorigin) is equivalent to setting the relevant `Header` in the `source` object. Declaratively configure cross-origin permissions for loaded images.
- [ ] [`decoding`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-decoding).
- [ ] `draggable`.
- [ ] [`fetchPriority`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-fetchpriority).
- [x] [`height`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-height) is the same as `source.height`.
- [ ] [`loading`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-loading).
- [x] [`referrerPolicy`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-referrerpolicy) is equivalent to setting the relevant `Header` in the `source` object. Declaratively configure referrer policy.
- [x] [`src`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-src) is the same as `source.uri`.
- [x] [`srcSet`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-srcset) is the same as setting a `source` array with `uri` and `scale` properties defined.
- [x] [`width`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-width) is the same as `source.width`.

## `<img>` events

Additional events for `<img>`.

- [ ] [`on-error`](https://developer.mozilla.org/en-US/docs/Web/API/Element/error_event)
- [ ] `on-load`

## `<input>` and `<textarea>` props

Additional props for `<input>` and `<textarea>`.

- [x] [`autoComplete`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/autocomplete) is the same as mapped values for existing `autoComplete` (Android) and `textContentType` (iOS).
- [ ] [`disabled`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/disabled).
- [x] [`enterKeyHint`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/enterkeyhint) is the same as mapped values for `returnKeyType`.
- [x] [`inputMode`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/inputmode)  is the same as mapped values for `keyboardType`.
  - [x] `inputMode === 'decimal'` is the same as `keyboardType = 'decimal-pad'`.
  - [x] `inputMode === 'email'` is the same as `keyboardType = 'email-address'`.
  - [x] `inputMode === 'none'` is the same as `showSoftInputOnFocus = false`.
  - [x] `inputMode === 'numeric'` is the same as `keyboardType = 'numeric'`.
  - [x] `inputMode === 'search'` is the same as `keyboardType = 'search'`.
  - [x] `inputMode === 'tel'` is the same as `keyboardType = 'phone-pad'`.
  - [x] `inputMode === 'url'` is the same as `keyboardType = 'url'`.
- [ ] [`max`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/max).
- [x] [`maxLength`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/maxlength).
- [ ] [`min`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/min).
- [ ] [`minLength`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/minlength).
- [ ] [`placeholder`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#placeholder).
- [x] [`readOnly`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#readonly) is the same as inverse `editable`.
- [ ] [`required`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#required).
- [x] [`rows`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/textarea#attr-rows) is the same as `numberOfLines` (`<textarea>` only).
- [ ] [`spellCheck`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/textarea#attr-spellcheck).
- [ ] [`value`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#value).

## `<input>` and `<textarea>` events

Additional events for `<input>` and `<textarea>`.

- [ ] `on-beforeinput`
- [ ] `on-change`
- [ ] `on-input`
- [ ] `on-invalid`
- [ ] `on-select`
- [ ] `on-selectionchange`

-----

# Styles API (CSS subset)

Significantly expanded styling capabilities to cover more of the features that are heavily relied upon by web engineers. Styles are used with the `css()` function and are passed to the `style` prop on elements.

## `css()` function

Styles must be wrapped in `css()`.

```js
<div style={css({ width: 100, color: props.color })} />
```

### `css.apply()` function

A function that returns native props, e.g., `const nativeProps = css.apply(...styles)`. On web, this is used to generate DOM `className` and `style` props, whereas on native it can be used to pass flattened/transformed styles and any other other native props to custom native components.

## CSS Compatibility

Existing properties that can be adjusted to align with the CSS spec.

* [x] [`aspectRatio`](https://developer.mozilla.org/en-US/docs/Web/CSS/aspect-ratio). Support string values, i.e., `'16 / 9'`, to align with CSS.
* [x] [`borderRadius`](https://developer.mozilla.org/en-US/docs/Web/CSS/border-radius). Support percentage values to align with CSS.
* [x] [`fontVariant`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-variant) support space-separated string values to align with CSS.
* [x] [`fontWeight`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-weight) support number values to align with React DOM / CSS.
* [x] [`objectFit`](https://developer.mozilla.org/en-US/docs/Web/CSS/object-fit) is equivalent to `resizeMode` for `<Image>`.
* [x] [`pointerEvents`](https://developer.mozilla.org/en-US/docs/Web/CSS/pointer-events) is equivalent to `pointerEvents` prop.
* [ ] [`position`](https://developer.mozilla.org/en-US/docs/Web/CSS/position). Support for `fixed` and `sticky` values.
* [x] [`transform`](https://developer.mozilla.org/en-US/docs/Web/CSS/transform). Support using string values to set transforms.
* [x] [`verticalAlign`](https://developer.mozilla.org/en-US/docs/Web/CSS/vertical-align) is equivalent to `textAlignVertical`.
* [x] [`userSelect`](https://developer.mozilla.org/en-US/docs/Web/CSS/user-select). Equivalent to using `selectable` prop on `<Text>`.

Existing logical properties that can be adjusted to adopt the CSS standard names. In addition, React Native will need to add native support for subtree-level writing direction controls. Setting the `dir` prop (or `direction` style) to `ltr` or `rtl` on an element should alter the way logical properties are resolved in the subtree.

* [ ] (`direction`. But it is [not recommended](https://w3c.github.io/csswg-drafts/css-writing-modes/#propdef-direction) on web, and we should consider removing it in favor of the `dir` prop.)
* [x] `borderEndEndRadius` is equivalent to `borderBottomEndRadius`.
* [x] `borderEndStartRadius` is equivalent to `borderBottomStartRadius`.
* [x] `borderStartEndRadius` is equivalent to `borderTopEndRadius`.
* [x] `borderStartStartRadius` is equivalent to `borderTopStartRadius`.
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
* [x] `inset` is equivalent to `top` & `bottom` & `right` & `left`.
* [x] `insetBlock` is equivalent to `top` & `bottom`.
* [x] `insetBlockEnd` is equivalent to `bottom`.
* [x] `insetBlockStart` is equivalent to `top`.
* [x] `insetInline` is equivalent to `right` & `left`.
* [x] `insetInlineEnd` is equivalent to `right` or `left`.
* [x] `insetInlineStart` is equivalent to `right` or `left`.

## CSS Animations

Support declarative keyframes and animations that can be optimized on the native side and avoid the need for `Animated`. Consider dispatching the corresponding W3C animation events too. See [animation](https://developer.mozilla.org/en-US/docs/Web/CSS/animation)).

- [ ] [`animationDelay`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-delay)
- [ ] [`animationDirection`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-direction)
- [ ] [`animationDuration`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-duration)
- [ ] [`animationFillMode`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-fill-mode)
- [ ] [`animationIterationCount`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-iteration-count)
- [ ] [`animationName`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-name)
- [ ] [`animationPlayState`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-play-state)
- [ ] [`animationTimingFunction`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-timing-function)

TBD: the relationship between `animationName` (or equivalent) and the API used to define Animation keyframes.

## CSS Colors

Support CSS 4 Colors, possibly by using [Colorjs.io](https://colorjs.io/) or implementing a native equivalent.

## CSS Container Queries

Prepare for [CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries).

## CSS Custom Properties

Support CSS custom property syntax `--variable-name`. This could be shimmed in user-space on top of the existing `StyleSheet` API, with a React Context used to provide variables and values to a subtree.

* [`CSS Variables`](https://w3c.github.io/csswg-drafts/css-variables/)
* [Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
* [`var()` function](https://developer.mozilla.org/en-US/docs/Web/CSS/var)

## CSS Filters

Support declarative filters as used by the CSS [filter](https://developer.mozilla.org/en-US/docs/Web/CSS/filter) style.

* [ ] [`blur()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/blur)
* [ ] [`brightness()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/brightness)
* [ ] [`contrast()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/contrast)
* [ ] [`drop-shadow()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/drop-shadow)
* [ ] [`grayscale()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/grayscale)
* [ ] [`hue-rotate()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/hue-rotate)
* [ ] [`invert()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/invert)
* [ ] [`opacity()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/opacity)
* [ ] [`saturate()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/saturate)
* [ ] [`sepia()`](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/sepia)

## CSS Functions

* [ ] [`calc()`](https://developer.mozilla.org/en-US/docs/Web/CSS/calc)
* [ ] [`clamp()`](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp)
* [ ] [`max()`](https://developer.mozilla.org/en-US/docs/Web/CSS/max)
* [ ] [`min()`](https://developer.mozilla.org/en-US/docs/Web/CSS/min)
* [ ] [`minmax()`](https://developer.mozilla.org/en-US/docs/Web/CSS/minmax)
* [ ] [`url()`](https://developer.mozilla.org/en-US/docs/Web/CSS/url)
* [ ] [`var()`](https://developer.mozilla.org/en-US/docs/Web/CSS/var)

## CSS Lengths

Support major units where supported by CSS.

* [ ] [`em`](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#relative_length_units) units.
* [ ] [`rem`](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#relative_length_units) units.
* [ ] `px` units.
* [ ] [`v*`](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#relative_length_units) units.
* [ ] [`%`](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Values_and_units#percentages) units.

## CSS Media Queries

Support CSS Media Queries.

Although Media Queries are not a preferred long-term solution for responsive design, [Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Container_Queries) are not yet widely supported by browsers. The dimensional Media Queries could be shimmed in user-space on top of the existing `StyleSheet` API.

Logical operators `not`, `and`, `or`, `only`.

[Media features](https://developer.mozilla.org/en-US/docs/Web/CSS/@media#media_features):

- [ ] [`aspect-ratio`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/aspect-ratio)
- [ ] [`forced-colors`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors)
- [ ] [`height`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/height)
- [ ] [`inverted-colors`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/inverted-colors)
- [ ] [`orientation`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/orientation)
- [ ] [`prefers-color-scheme`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [ ] [`prefers-contrast`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-contrast)
- [ ] [`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
- [ ] [`resolution`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/resolution)
- [ ] [`width`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/width)

Proposed syntax:

```js
// Either
const styles = StyleSheet.create({
  root: {
    position: 'absolute',
    '@media (max-width: 600px)': {
      position: 'sticky',
    }
  }
});

// Or
const styles = StyleSheet.create({
  root: {
    position: {
      default: 'absolute',
      '@media (max-width: 600px)': 'sticky',
    }
  }
});
```

The benefit of the latter is clearer expectations (and greater constraints) about when properties are overridden. For example, it's not as clear what the value for `position` would be in the following:

```js
const styles = StyleSheet.create({
  root: {
    position: 'absolute',
    '@media (max-width: 600px)': {
      position: 'sticky',
    }
  },
  override: {
    position: 'relative',
  }
});

const style = [ styles.root, styles.override ];
// {
//   position: relative,
//   '@media (max-width: 600px)': {
//     position: 'sticky',
//   }
// }
```

Whereas in the next example we can set expectations that a property *and any previous conditions* are completed overridden by any declaration that later modifies the property in any way.

```js
const styles = StyleSheet.create({
  root: {
    position: {
      default: 'absolute',
      '@media (max-width: 600px)': 'sticky'
    }
  },
  override: {
    position: {
      default: 'relative',
    }
  }
});

const style = [ styles.root, styles.override ];
// {
//   position: relative
// }
```

Consideration is also required to determine what the final value of a property is when Media Query conditions overlap, i.e., which media features and which conditions (e.g., multiple `width` conditions) take priority over others.

## CSS Properties and Values

Miscellaneous CSS properties and values that should be supported on native.

* [ ] [`backgroundImage`](https://developer.mozilla.org/en-US/docs/Web/CSS/background-image). Add support for setting background images via `url()`. Stretch: support CSS gradients.
* [ ] [`backgroundOrigin`](https://developer.mozilla.org/en-US/docs/Web/CSS/background-origin).
* [ ] [`backgroundPosition`](https://developer.mozilla.org/en-US/docs/Web/CSS/background-position).
* [ ] [`backgroundRepeat`](https://developer.mozilla.org/en-US/docs/Web/CSS/background-repeat).
* [ ] [`backgroundSize`](https://developer.mozilla.org/en-US/docs/Web/CSS/background-size).
* [ ] [`boxSizing`](https://developer.mozilla.org/en-US/docs/Web/CSS/box-sizing).
* [ ] [`boxShadow`](https://developer.mozilla.org/en-US/docs/Web/CSS/box-shadow). Add native support for CSS box shadows to replace buggy, iOS-specific `shadow*` styles. Resolves issue (#26110)[https://github.com/facebook/react-native/pull/26110] and [multiple other issues](https://github.com/facebook/react-native/search?q=ios+shadow&type=issues).
* [ ] [`caretColor`](https://developer.mozilla.org/en-US/docs/Web/CSS/caret-color).
* [ ] [`clipPath`](https://developer.mozilla.org/en-US/docs/Web/CSS/clip-path).
* [ ] [`display`](https://developer.mozilla.org/en-US/docs/Web/CSS/display) values of `block`, `contents` (resolves proposal [#348](https://github.com/react-native-community/discussions-and-proposals/issues/348)), `inline`, `inline-block`, `inline-flex`.
* [ ] [`float`](https://developer.mozilla.org/en-US/docs/Web/CSS/float).
* [ ] [`justifySelf`](https://developer.mozilla.org/en-US/docs/Web/CSS/justify-self). Sets the way a box is justified inside its alignment container along the appropriate axis.
* [ ] ([`lineClamp`](https://www.w3.org/TR/css-overflow-3/#propdef-line-clamp) is equivalent to `numberOfLines` prop on `<Text>` components).
* [ ] [`objectPosition`](https://developer.mozilla.org/en-US/docs/Web/CSS/object-position).
* [ ] [`overflowBlock`](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-block).
* [ ] [`overflowInline`](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-inline).
* [ ] [`overflowX`](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-x).
* [ ] [`overflowY`](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow-y).
* [ ] [`placeContent`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-content). Shorthand for setting `alignContent` and `justifyContent`. Value is an enum of `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `right`, `safe`, `space-around`, `space-between`, `space-evenly`, `start`, `stretch`, `unsafe`. Note that `space-evenly` should also be supported for `alignContent` and `justifyContent`.
* [ ] [`placeItems`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-items). Shorthand for setting `alignItems` and `justifyItems`. Value is an enum of `auto`, `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `right`, `start`, `stretch`.
* [ ] [`placeSelf`](https://developer.mozilla.org/en-US/docs/Web/CSS/place-self). Shorthand for setting `alignSelf` and `justifySelf`. Value is an enum of `auto`, `baseline`, `first baseline`, `last baseline`, `center`, `end`, `flex-start`, `flex-end`, `left`, `normal`, `right`, `start`, `stretch`.
* [ ] [`rotate`](https://developer.mozilla.org/en-US/docs/Web/CSS/rotate).
* [ ] [`scale`](https://developer.mozilla.org/en-US/docs/Web/CSS/scale).
* [ ] [`scrollSnap*`](https://drafts.csswg.org/css-scroll-snap-1/#overview)
* [ ] `textShadow`. Add native support for CSS text shadows. Resolves issue (#26110)[https://github.com/facebook/react-native/pull/26110].
* [ ] [`touchAction`](https://developer.mozilla.org/en-US/docs/Web/CSS/touch-action).
* [ ] [`transformStyle`](https://developer.mozilla.org/en-US/docs/Web/CSS/transform-style). Resolves proposal [#425](https://github.com/react-native-community/discussions-and-proposals/issues/425).
* [ ] [`translate`](https://developer.mozilla.org/en-US/docs/Web/CSS/translate).
* [ ] [`visibility`](https://developer.mozilla.org/en-US/docs/Web/CSS/visibility). Add support for visually hiding elements.
* [ ] [`whiteSpace`](https://developer.mozilla.org/en-US/docs/Web/CSS/white-space).

## CSS Transitions

Support declarative transitions that can be optimized on the native side and avoid the need for `Animated`. Consider dispatching the corresponding W3C transition events too.

- [ ] [`transitionDelay`](https://developer.mozilla.org/en-US/docs/Web/CSS/transition-delay)
- [ ] [`transitionDuration`](https://developer.mozilla.org/en-US/docs/Web/CSS/transition-duration)
- [ ] [`transitionProperty`](https://developer.mozilla.org/en-US/docs/Web/CSS/transition-property)
- [ ] [`transitionTimingFunction`](https://developer.mozilla.org/en-US/docs/Web/CSS/transition-timing-function)
