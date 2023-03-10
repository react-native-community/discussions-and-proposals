---
title: DOM traversal and layout APIs in React Native
author: Rubén Norte
date: 2023-03-01
updated: 2023-03-10
---

# RFC: DOM traversal and layout APIs in React Native

## Summary

This is a proposal to allow users to access a representation of the native view hierarchy in React Native using DOM-compatible APIs.

The main goals are:

1. Allowing imperative access to the current state of the UI to unblock advanced use cases (see [⚓️ Motivation](#motivation)).
2. Simplifying the reuse of Web libraries that implement solutions for them.

The scope of this proposal is restricted to:

* Tree structure (traversing nodes) and layout information.
* **Read-only** APIs. Mutations and imperative actions (like `scroll` and `focus`) are out of the scope of this proposal.
* Apps using the new React Native renderer (Fabric).

__*This proposal is a subset of [*RFC: React DOM for Native*](https://github.com/react-native-community/discussions-and-proposals/pull/496), which has a larger scope at aligning with Web standards. This focuses on bringing part of the `Node` and `Element` APIs to provide access to the Host Tree in React Native.*__

## Basic Example

```javascript
function Component(props) {
  const ref = useRef();
  const firstChildRef = useRef();
  const secondChildRef = useRef();
  const textRef = useRef();

  useEffect(() => {
    const node = ref.current;
    const firstChildElement = firstChildRef.current;
    const secondChildElement = secondChildRef.current;
    const textNode = textRef.current;

    // Accessing parent, children, siblings...
    node.parentNode;
    node.childNodes;
    firstChildElement.nextSibling;
    node.getRootNode();

    // We have referential equality as on the Web
    node.firstChild === firstChildElement;
    firstChildElement.nextSibling === secondChildElement;
    secondChildElement.parentNode === node;
    node.getRootNode() === firstChildElement.getRootNode();

    // We can access existing methods in those nodes
    node.focus();

    // And the new properties and methods added in this proposal, e.g.:
    const rect = firstChildElement.getBoundingClientRect();
    const offsetTop = firstChildElement.offsetTop;
    const scrollHeight = firstChildElement.scrollHeight;
    const tagName = firstChildElement.tagName; // 'View'

    // Accessing text data and nodes
    const textContent = textNode.textContent; // 'Some text'
    const textContentAlt = textNode.firstChild.data; // 'Some text'

    // In the future (NOT PART OF THIS PROPOSAL)
    const intersectionObserver = new IntersectionObserver((entries) => {});
    intersectionObserver.observe(node);
  }, []);

  return (
    // If these were <div>, this would work as-is on the Web
    <View ref={ref}>
      <View ref={firstChildRef}>
        {props.children}
        <Text ref={textRef}>Some text</Text>
      </View>
      <View ref={secondChildRef}>
        {props.children}
      </View>
    </View>
  );
}
```

## Motivation

On the Web, most product code that needs to use refs to access DOM nodes does it to call imperative methods (like `focus()` or `blur()`) or to access layout information (e.g.: to calculate derived styles based on them). This is also true in React Native, and the existing methods defined on native component refs already support this use case.

There are other use cases, which we could categorize as advanced, that require a deeper understanding of the structure of the tree. E.g.:

* Performance instrumentation: some libraries use APIs like `MutationObserver` and `IntersectionObserver`, together with DOM traversal APIs, to track UI changes on the page in order to compute performance metrics (e.g.: last visual change in an interaction).
* Debugging: having full access to the host tree and its layout would allow taking lightweight snapshots of the UI in regular intervals or after specific changes, which would be useful for debugging in tests and reported errors in production.

These use cases require being able to inspect the full UI tree imperatively, reference nodes in arbitrary positions of the tree, etc. This is not currently possible in React Native without writing native code (which most times is platform-specific). We want to make it easier to share code across platforms (including Web) and this simplifies that significantly.

Additionally, React is considering new features that would have a close integration with the host platform. E.g.:

* Fragment references. These would be a custom node instance handled by React as a proxy for the contents of the fragment, which would have a similar API as `Element` on the Web.
* A new option for `Offscreen` to mark its content as visible or hidden depending on their actual visibility on the viewport. On the Web, this would use a reference to a node (potentially a fragment) and the `IntersectionObserver` API to detect visibility.

Providing this kind of access to the host tree on the Web and on all React Native platforms would speed up the rollout of these features, reducing the existing differences between React DOM and React Native.

## Detailed Design

### API Reference

The existing [`ReactFabricHostComponent`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L135) class, which is instantiated for all host elements and passed to user code through refs, would be renamed as `ReactNativeElement`. This class would extend the `ReadOnlyNode` and `ReadOnlyElement` classes. All these classes would be defined in the global scope for symmetry with the Web.

The properties and methods defined in these classes would have similar semantics as on the Web, except for:

* Setters for properties that are writable on the Web and not defined or read-only on React Native. They would be either a noop with a warning when assigned, or they would throw an exception if in strict mode (marked with ⚠️/💣 in the list below).
* Getters for properties that are discouraged because of loss of precision, like `clientWidth` and `offsetHeight`. They would be implemented in a spec-compliant way but they would log a warning encouraging users to use a different API like `getBoundingClientRect` with better precision (marked with ⚠️ in the list below).

This is a reference of the API provided by those classes:

* [`Node`](https://developer.mozilla.org/en-US/docs/Web/API/Node) (implemented as `ReadOnlyNode`):
    * Static properties:
        * [Node type constants](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType#value):
            * `ELEMENT_NODE (1)`
            * `ATTRIBUTE_NODE (2)` (unused in this proposal)
            * `TEXT_NODE (3)`
            * `CDATA_SECTION_NODE(4)` (unused in this proposal)
            * `PROCESSING_INSTRUCTION_NODE (7)` (unused in this proposal)
            * `COMMENT_NODE (8)` (unused in this proposal)
            * `DOCUMENT_NODE (9)` (unused in this proposal)
            * `DOCUMENT_TYPE_NODE (10)` (unused in this proposal)
            * `DOCUMENT_FRAGMENT_NODE (11)` (unused in this proposal)
        * [Document position constants](https://developer.mozilla.org/en-US/docs/Web/API/Node/compareDocumentPosition#return_value):
            * `DOCUMENT_POSITION_DISCONNECTED (1)`
            * `DOCUMENT_POSITION_PRECEDING (2)`
            * `DOCUMENT_POSITION_FOLLOWING (4)`
            * `DOCUMENT_POSITION_CONTAINS (8)`
            * `DOCUMENT_POSITION_CONTAINED_BY (16)`
            * `DOCUMENT_POSITION_IMPLEMENTATION_SPECIFIC (32)` (unused in this proposal)
    * Instance properties:
        * `get `[`childNodes`](https://developer.mozilla.org/en-US/docs/Web/API/Node/childNodes)
        * `get `[`firstChild`](https://developer.mozilla.org/en-US/docs/Web/API/Node/firstChild)
        * `get `[`isConnected`](https://developer.mozilla.org/en-US/docs/Web/API/Node/isConnected) (indicates if the node is currently mounted)
        * `get `[`lastChild`](https://developer.mozilla.org/en-US/docs/Web/API/Node/lastChild)
        * `get `[`nextSibling`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nextSibling)
        * `get `[`nodeName`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeName) (name to use in built-in RN components is TBD)
        * `get `[`nodeType`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType) (always `Node.ELEMENT_NODE (1)`)
        * `get `[`nodeValue`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeValue) (always `null`)
        * `get `[`parentElement`](https://developer.mozilla.org/en-US/docs/Web/API/Node/parentElement) (alias for [`parentNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node/parentNode), as text nodes cannot have children)
        * `get `[`parentNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node/parentNode)
        * `get `[`previousSibling`](https://developer.mozilla.org/en-US/docs/Web/API/Node/previousSibling)
        * `get `[`textContent`](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent)
        * ⚠️/💣 `set `[`nodeValue`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeValue) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣 `set `[`textContent`](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent)  (noop with a warning, or exception if in strict mode)
    * Instance methods:
        * [`compareDocumentPosition()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/compareDocumentPosition)
        * [`contains()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/contains)
        * [`getRootNode()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode)
        * [`hasChildNodes()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/hasChildNodes)
* [`Element`](https://developer.mozilla.org/en-US/docs/Web/API/Element) (implemented as `ReadOnlyElement`):
    * Instance properties:
        * `get `[`childElementCount`](https://developer.mozilla.org/en-US/docs/Web/API/Element/childElementCount)
        * `get `[`children`](https://developer.mozilla.org/en-US/docs/Web/API/Element/children)
        * ⚠️ `get `[`clientHeight`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientHeight) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️ `get `[`clientLeft`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientLeft) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️ `get `[`clientTop`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientTop) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️ `get `[`clientWidth`](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientWidth) (spec-compliant implementation with warning about the loss of precision)
        * `get `[`firstElementChild`](https://developer.mozilla.org/en-US/docs/Web/API/Element/firstElementChild)
        * `get `[`id`](https://developer.mozilla.org/en-US/docs/Web/API/Element/id)
        * `get `[`lastElementChild`](https://developer.mozilla.org/en-US/docs/Web/API/Element/lastElementChild)
        * `get `[`nextElementSibling`](https://developer.mozilla.org/en-US/docs/Web/API/Element/nextElementSibling)`
        * `get `[`previousElementSibling`](https://developer.mozilla.org/en-US/docs/Web/API/Element/previousElementSibling)
        * `get `[`scrollHeight`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollHeight)
        * `get `[`scrollLeft`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollLeft)
        * `get `[`scrollTop`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop)
        * `get `[`scrollWidth`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollWidth)
        * `get `[`tagName`](https://developer.mozilla.org/en-US/docs/Web/API/Element/tagName) (alias for [`nodeName`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeName))
        * ⚠️/💣 `set `[`className`](https://developer.mozilla.org/en-US/docs/Web/API/Element/className) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣 `set `[`id`](https://developer.mozilla.org/en-US/docs/Web/API/Element/id) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣 `set `[`innerHTML`](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣 `set `[`outerHTML`](https://developer.mozilla.org/en-US/docs/Web/API/Element/outerHTML) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣  `set `[`scrollLeft`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollLeft) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣  `set `[`scrollTop`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop) (noop with a warning, or exception if in strict mode)
    * Instance methods:
        * [`getBoundingClientRect()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect)
        * [`getClientRects()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getClientRects)
* `ReactNativeElement`, with the following definition from [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement):
    * Instance properties:
        * ⚠️ `get `[`offsetHeight`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetHeight) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️ `get `[`offsetLeft`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetLeft) (spec-compliant implementation with warning about the loss of precision)
        * [`offsetParent`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetParent) (alias for [`parentNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node/parentNode) as all elements are positioned relative to their parents in React Native, or `null` if the element is not displayed)
        * ⚠️ `get `[`offsetTop`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetTop) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️ `get `[`offsetWidth`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/offsetWidth) (spec-compliant implementation with warning about the loss of precision)
        * ⚠️/💣 `set `[`innerText`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/innerText) (noop with a warning, or exception if in strict mode)
        * ⚠️/💣 `set `[`outerText`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/outerText) (noop with a warning, or exception if in strict mode)

Text ***nodes*** (not to be confused with text elements), would be instances of [`Text`](https://developer.mozilla.org/en-US/docs/Web/API/Text) with the following definition:

* [`Node`](https://developer.mozilla.org/en-US/docs/Web/API/Node):
    * Same as for `ReactNativeElement` before, except all methods accessing children return `null` or an empty list, as text nodes cannot have children.
    * [`nodeType`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeType) (always `Node.TEXT_NODE (3)`)
    * `get `[`nodeValue`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeValue) (always the underlying text)
    * ⚠️/💣 `set `[`nodeValue`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeValue) (noop with a warning, or exception if in strict mode)
* [`CharacterData`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData):
    * Instance properties:
        * `get `[`data`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/data)
        * `get `[`length`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/length)
        * `get `[`nextElementSibling`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/nextElementSibling)
        * `get `[`previousElementSibling`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/previousElementSibling)
        * ⚠️/💣 `set `[`data`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/data) (noop with a warning, or exception if in strict mode)
    * Instance methods:
        * [`substringData()`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData/substringData)
* [`Text`](https://developer.mozilla.org/en-US/docs/Web/API/Text):
    * *No additional properties or methods*

### Semantics

#### Tree representation

In React Native, the following React fragment:

```html
<>
  <Text>Some text</Text>
  <View>
    <View>
      <Text>another text</Text>
    </View>
  <View />
</>
```

Would be represented with the following structure in the host tree:

```
Root element
  ├ Text element
  │  └ Text node ('Some text')
  ├ View element
  │  └ View element
  │    └ Text element
  │      └ Text node ('another text')
  └ View element
```

On the Web, most of the nodes of the tree are subclasses of [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) (like [`HTMLDivElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDivElement)) or [`Text`](https://developer.mozilla.org/en-US/docs/Web/API/Text) nodes.

On React Native, all nodes would be:

* **Elements rendered by native components**, which would be instances of **`ReactNativeElement`**.
    * This class would inherit from [`ReadOnlyNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node) and [`ReadOnlyElement`](https://developer.mozilla.org/en-US/docs/Web/API/Element), which would exist in the global scope at runtime.
    * [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) would not be defined directly in React Native and its methods would be implemented in the `ReactNativeElement` class instead. We could define `HTMLElement` and its subclasses later (potentially as `ReadOnlyHTMLElement`), if we added a compatibility layer with the DOM, supporting HTML elements like `<div>` and `<span>`.
* Or **text nodes**, which would be instances of [**`ReadOnlyText`**](https://developer.mozilla.org/en-US/docs/Web/API/Text) class (as on the Web, with the same class hierarchy: [`ReadOnlyText`](https://developer.mozilla.org/en-US/docs/Web/API/Text) > [`ReadOnlyCharacterData`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData) > [`ReadOnlyNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node)). These classes about also be exposed in the global scope.

In this tree representation we would only have nodes for views created and managed by React Native. This means that if we have a native UI that renders a React Native component only under a specific root view, the contents outside of the React Native root view would not be accessible.

#### Root elements

Root elements would be special elements representing the root of a specific tree. React Native does not require users to specify a single element as the root of the tree (as the Web does with [`document`](https://developer.mozilla.org/en-US/docs/Web/API/Document) and the [`documentElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/documentElement) property that points to the root [`<html>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) element), but it does create one internally. Calling [`getRootNode()`](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode) on a node would return a reference to the root element in the current tree. This root element would be an instance of `ReactNativeElement`.

This proposal does not address the direct access to any of the trees. Instead, all accesses would be done through refs passed to native components.

#### Text nodes

Text nodes exist in React Native in a similar way as they exist on the Web. For example, these elements:

```html
<Text>
  You can nest
  <Text>text content</Text>
  in nested text components
</Text>
```

Would be represented as a tree with the following structure:

```
Root element
  └ Text element
    ├ Text node ('You can nest')
    ├ VirtualText element
    │  └ Text node ('text content')
    └ Text node ('in nested text components')
```

Text ***elements*** are accessible through refs passed to built-in [`Text`](https://reactnative.dev/docs/text) component or traversing parents/children. As the rest of built-in components, they are instances of the `ReactNativeElement` class.

Text ***nodes*** are only accessible as child nodes of [`Text`](https://reactnative.dev/docs/text) (or `VirtualText`) elements (through APIs like [`childNodes`](https://developer.mozilla.org/en-US/docs/Web/API/Node/childNodes) or [`firstChild`](https://developer.mozilla.org/en-US/docs/Web/API/Node/firstChild)). They are exposed as instances of a class implementing the [`ReadOnlyText`](https://developer.mozilla.org/en-US/docs/Web/API/Text), [`ReadOnlyCharacterData`](https://developer.mozilla.org/en-US/docs/Web/API/CharacterData) and [`ReadOnlyNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node) interfaces specified before.

As on the Web:

* Text nodes cannot have children.
* All APIs referencing elements, instead of nodes, would ignore text nodes.

#### Disconnected nodes

On the Web, removing an element from the tree does not affect the structure of its descendants. If you had a `<ul>` element with multiple `<li>` children and you removed the `<ul>` element from its parent, it would still have all the `<li>` as children.

In React Native, mutations are not imperative but done through the React reconciler. We could potentially have the same behavior as on the Web and keep the structure of disconnected nodes but that would provide little value and would increase the complexity of implementing potential features like view re-parenting (which is not on the roadmap).

Instead of replicating that behavior, we would just **assume that disconnected nodes no longer have a structure**. They would have **no parents and no children**.

#### Consistency and updates

On the Web, the DOM tree is a live representation of the current state of the UI. As JavaScript and rendering happens on the same thread, changes to the DOM are always synchronous. There are only a few exceptions to this, like passive effects, where the values accessible in JavaScript events do not represent the current state of the UI (e.g.: scroll position does not update synchronously).

In React Native, the host tree is a representation of the last state of the UI known by the renderer. Changes to the actual UI can come from JavaScript (when do initial render or apply state updates), or from the UI thread (e.g.: when we scroll).

When **UI changes are triggered from the main thread** (e.g.: when we scroll):

1. We create a **new representation of the host tree**. This is known as applying a native state update.
2. We **commit that tree asynchronously, synchronized with the JavaScript thread.**

This means that JavaScript always has a consistent view of the host tree (the host tree is never mutated in the middle of the execution of a JavaScript function) but as that view has been committed asynchronously, it is not necessarily the last state of the UI (only the last state known by the renderer).

![Diagram describing how asynchronous native state updates are currently dispatched from the UI thread to the JS thread](/assets/native-state-updates.png?raw=true) *Synchronization between JavaScript and native state updates (worst case scenario)*

When **UI changes are triggered from JavaScript, the host tree is committed synchronously**. The last known state is applied to the committed tree (which is known as progressing state) but that committed tree is still not necessarily synchronized with the UI thread.

Given these constraints, we can ensure that **product logic in JavaScript would always have a consistent view of the host tree.** This logic can only exist:

1. **In events** (e.g.: user events, timers or network callbacks). Event handlers would see an immutable version of the tree during its execution. If event handlers trigger state updates, these would always be processed by React after the execution of the handler.
2. **During render**. In this case, the version of the tree that this code would see is the one that matches the last commit (done from React or native). This use case is not recommended in React DOM and given we control the definition of the nodes in the tree, **we can detect it and warn about it at runtime**.
3. **In effects**. In this case, the version of the tree would match what was committed with that effect. Layout effects can trigger new state updates, but those would also happen after the effects have executed, keeping the consistency that we see when executing event handlers.

This example shows how the observed state would be updated:

```javascript
const ref = useRef();

useLayoutEffect(() => {
  const node = ref.current;
  node.scrollTop; // 0
  node.scrollTop; // 0
  node.scrollTop; // 0

  setTimeout(() => {
    node.scrollTop; // 123
    node.scrollTop; // 123
    node.scrollTop; // 123
  }, 1000);

  return () => {
    node.scrollTop; // 800
    node.scrollTop; // 800
    node.scrollTop; // 800
  };
}, []);

const onPress = useCallback(() => {
  const node = ref.current;
  node.scrollTop; // 200
  node.scrollTop; // 200
  node.scrollTop; // 200

  // Imperative updates are not synchronous in React Native.
  // They are observable after the host tree is updated,
  // respecting the consistency guarantees defined in the proposal.
  // On the Web, this would only be asynchronous
  // when using the 'smooth' behavior option.
  node.scrollTo(0, 1000);

  node.scrollTop; // 200
  node.scrollTop; // 200
  node.scrollTop; // 200
}, []);

return (
  <ScrollView ref={ref}>
    <Pressable onPress={onPress}>
        <Text>Hello world!</Text>
    </Pressable>
  </ScrollView>
);
```

Because of the current constraints, **the values exposed through these new APIs would be consistent but not necessarily up to date**. The tree structure (existing nodes) would be, because they are never mutated from the UI thread, but layout information might not be.

All events in React Native are asynchronous (regarding the UI thread). They are what we would call passive events on the Web. If **we added support for synchronous events** (which would be dispatched to JavaScript blocking the UI thread), **we could ensure that these values are both consistent and up to date**. We could also introduce other synchronization mechanisms to force this outside of user events (e.g.: using `requestAnimationFrame`). If and when we do that, we could introduce **warnings when accessing these APIs outside of these synchronized accesses**.

![Diagram describing the synchronization of the UI thread and the JS thread when using synchronous events](/assets/sync-events.png?raw=true)
*Potential solution with synchronous/active events*

#### Animations

There are currently 2 [built-in animation systems in React Native](https://reactnative.dev/docs/animations):

* **Layout Animations**: when committing a new tree, this system handles the transitions between the previous commit and the new one. The committed tree represents the end state of the UI and the transitions are handled directly from the host platform. In this case **JavaScript code would observe the end state** when accessing DOM APIs during a transition.
* **Animated API**: this system allows users to specify animations declaratively when rendering components. The way these animations are performed depend on the `useNativeDriver` flag.
    * When **using the native driver**, we commit the initial state of the UI as a shadow tree, and an external system runs the animations directly on the host platform (without reflecting those in the shadow tree). In that case, **JavaScript code would only observe the initial state**. For JavaScript to observe the final state, users would have to register a callback by the end of the animation and do a state update with the final values.
    * When using **JS-driven animations**:
        * **Currently**, we commit a new tree for each animation tick from JavaScript. In that case, **all intermediate states would be observable from JavaScript**.
        * **In the near future**, that is going to change to use `setNativeProps` instead, which means that animations will bypass the shadow tree and **work similarly as animations using the native driver**. Only the initial state would be observable in this case.

With the current animation systems, the observable state from JavaScript would likely be out of sync, but still consistent within the same event or effect phase. We need to make sure this is communicated effectively to users.

**Reanimated** is a widely used library for animations in React Native. In that case, all **animation ticks are defined in the shadow tree, but updates are not synchronized with the JavaScript thread**. That means the consistency guarantees defined in this proposal are not respected, so it would be possible to observe inconsistent values in the same JavaScript task. They also extend certain core libraries in order to implement this animation libraries (like the `UIManager` binding in JavaScript), as we currently do not provide extension points to allow this. We would have to verify that their extensions continue working after we make these changes.

These animation systems have known problems that do not seem solvable with incremental changes, and there have been talks about building a new system (integrated into React) to solve them. This new system could also solve the synchronization problems mentioned here. For example, it could force the animation state to be flushed to the shadow tree when executing JavaScript code that is synchronized with the UI thread, like synchronous events.

#### Live collections

In this proposal, only 2 methods return a collection of nodes or elements: [`Node.prototype.childNodes`](https://developer.mozilla.org/en-US/docs/Web/API/Node/childNodes) and [`Element.prototype.children`](https://developer.mozilla.org/en-US/docs/Web/API/Element/children). On the Web, they are defined as “live collections”, which means that their contents update as the underlying DOM tree changes.

This behavior would be intentionally **not implemented in React Native**, where those collections would have the same API as on the Web (being array-like structures but not array instances, etc.) but would not update automatically. They would be implemented as getters and the result would be computed only when the getter is invoked.

#### Existing host node methods

[`ReactFabricHostComponent`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L135) already implements the following methods:

* `blur`
* `focus`
* `measure`
* `measureInWindow`
* `measureLayout`
* `setNativeProps`

These **would be kept** in `ReactNativeElement` **for backward compatibility reasons**, but we could introduce a warning to encourage users to migrate to the new APIs (so we can deprecate them in the future). Among those, only [`blur`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/blur) and [`focus`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) match existing DOM APIs, although [`focus`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/focus) does not support options. This proposal does not contemplate any changes to those methods, which would be kept unmodified in the new `ReactNativeElement` class.

### Implementation

#### API implementation

The new [`ReadOnlyNode`](https://developer.mozilla.org/en-US/docs/Web/API/Node), [`ReadOnlyElement`](https://developer.mozilla.org/en-US/docs/Web/API/Element) and `ReactNativeElement` classes would be defined in the React Native repository. Their properties and methods would invoke methods on `UIManager`, with a reference to the shadow node they represent. E.g.:

```javascript
class Node {
  get childNodes() {
    return createArrayLike(
      nativeFabricUIManager.getChildNodes(
        getShadowNode(this),
      ),
    );
  }
}
```

At the moment, [`ReactFabricHostComponent`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L135) is defined in the React repository. We would move it to React Native before its refactor as the new class hierarchy. In React, we would create instances of the new `ReactNativeElement` class through a new method in [`ReactNativePrivateInterface`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L39), which is already being used to access other parts of React Native, e.g.:

In [`ReactFabricHostConfig`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L389-L394), we’d refactor:

```javascript
  const component = new ReactFabricHostComponent(
    tag,
    viewConfig,
    props,
    internalInstanceHandle,
  );
```

Into something like this:

```javascript
  const component = ReactNativePrivateInterface.createReactNativeElement(
    tag,
    viewConfig,
    props,
    internalInstanceHandle,
  );
```

This would allow us to encapsulate the definition of the new methods and would allow us to iterate on them faster, without requiring pulling new version of React into React Native.

#### State consistency handling

Implementing new state consistency mechanisms would not be necessary because all current updates to the tree are correctly synchronized with the execution of JavaScript code, following the semantics we defined in this proposal.

#### Root elements

[`RootShadowNode`](https://github.com/facebook/react-native/blob/main/ReactCommon/react/renderer/components/root/RootShadowNode.cpp) exists in React Native but it is not currently exposed to JavaScript and [it does not have an associated instance handle](https://github.com/facebook/react-native/blob/main/ReactCommon/react/renderer/mounting/ShadowTree.cpp#L244). This orchestration would have to be added to the integration between React and React Native.

#### Text nodes

Text nodes are not currently accessible in React Native, as nodes are only exposed through refs and refs can only be used in elements, but they already exist and they are also [handled by React](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L402). As part of the host configuration in React, we can define how we are going to create text instances. The current implementation just [creates a wrapper object pointing to the node in the shadow tree](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L402), but we can refactor it to align with the current implementation of host components, which also creates an instance of the [`ReactFabricHostComponent`](https://github.com/facebook/react/blob/main/packages/react-native-renderer/src/ReactFabricHostConfig.js#L135) class.

### Dependencies

This proposal depends on fixing the semantics for layout effects in React Native. At the moment, layout is asynchronous and when layout effects execute layout information is likely not computed yet. We are working on a fix for this by doing layout synchronously in the JavaScript thread. Doing that would fix layout effects and would preserve the semantics specified by this API.

### Limitations

This proposal intentionally **excludes certain concepts and APIs**, including some properties and methods from [`Node`](https://developer.mozilla.org/en-US/docs/Web/API/Node), [`Element`](https://developer.mozilla.org/en-US/docs/Web/API/Element) and [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement):

* All operations that manipulate the structure of the tree or the attributes of the nodes. This is unlikely to change in future proposals. There are 2 main reasons for this:
    * Complexity/performance: the React Native renderer works with immutable UI trees and providing an interface that allows mutations would require to either have a new representation of the tree with the pending changes to be applied later (complexity), or flushing changes immediately as they happen (performance).
    * Consistency: making changes to the UI hierarchy outside of React risks creating inconsistency issues where the information that React has about the tree does not match the state of the tree. This is a risk on the Web and we want to avoid it completely in React Native.
* The definition of [`document`](https://developer.mozilla.org/en-US/docs/Web/API/Document) or access to it through APIs like [`node.ownerDocument`](https://developer.mozilla.org/en-US/docs/Web/API/Node/ownerDocument). The [`document`](https://developer.mozilla.org/en-US/docs/Web/API/Document) API is very coupled to Web and many of the concepts it defines do not currently apply to React Native (`head` and `body` elements, `contentType`, `documentURI`, etc.). If a similar concept is implemented in React Native, it would be part of a separate proposal that considers these constraints.
* Access to attributes ([`getAttribute()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAttribute), [`getAttributeNames()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAttributeNames), etc.). These require additional work to define a public API for the available attributes for each native component. All built-in components in React Native are actually wrappers on top of actual native components, so it is unclear what are the attributes these elements should be exposing (Should they be the props passed to the built-in component? Should they be the processed props passed to the native component?). This proposal avoids this definition to unblock the existing use cases. If implemented, it would be part of a future proposal.
* The concept of classes and any properties and methods that operate on them ([`className`](https://developer.mozilla.org/en-US/docs/Web/API/Element/className), [`classList`](https://developer.mozilla.org/en-US/docs/Web/API/Element/classList), etc.). React Native does not support classes for styling, which is one of the main use cases for them on the Web.
* All properties and methods inherited from [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) ([`addEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener), [`dispatchEvent`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent), [`removeEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/removeEventListener)), and any other methods that handle events. These would be covered in a separate proposal.
* Document query APIs or APIs using selectors (e.g.: `querySelector()`, [`getElementById()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/getElementById), `getElementsBy*()`, [`closest()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/closest)). These require access to attributes, classes, etc. which do not currently exist in React Native, and require significant complexity to implement efficiently.

Additionally, the features and APIs described in this proposal would **only be available in Fabric**, the React Native native renderer in the new architecture. The main reason is that we rely on an intermediate representation of the UI tree that only Fabric has, as the previous renderer (Paper) performance mutations directly on the host platform and does not have a mechanism to access information from it synchronously (which we require to implement the synchronous methods defined in this proposal).

### Code sharing and feature detection

One of the goals of this proposal is to make it easier to share code between Web and React Native. Initially, most of the code that would be able to be shared would be hooks and external functions, not components and rendering code directly. E.g.:

```javascript
function FirstReactNativeComponent() {
  const ref = useCrossPlatformHook();
  return <View ref={ref} />;
}

function FirstWebComponent() {
  const ref = useCrossPlatformHook();
  return <div ref={ref} />;
}

function SecondReactNativeComponent() {
  const ref = useRef();

  useLayoutEffect(() => {
    crossPlatformFunctionUsingNodes(ref.current);
  }, []);

  return <View ref={ref} />;
}

function SecondWebComponent() {
  const ref = useRef();

  useLayoutEffect(() => {
    crossPlatformFunctionUsingNodes(ref.current);
  }, []);

  return <div ref={ref} />;
}
```

In these cases, both `useCrossPlatformHook` and `crossPlatformFunctionUsingNodes` should be able to know what type of node they’re receiving and whether certain operations are supported or not.

These are some types of detections they should be able to use:

* Checking if a Web-only property is defined (via feature detection):

```javascript
function crossPlatformFunctionUsingNodes(node) {
  if (node.innerHTML != null) {
    // they can use innerHTML!
  }

  if (node.getAttribute != null) {
    // they can access attributes!
  }
}
```

* Checking if a reference is a node or an element. The distinction between nodes and elements exist in both environments and some of the methods on elements are available on both environments but not on nodes (this would be feature inference, which is discouraged over feature detection):

```javascript
function crossPlatformFunctionUsingNodes(node) {
  if (node.nodeType === Node.`ELEMENT_NODE`) {
    // they can use node.children and node.parentElement
  }
}
```

* Checking if a property is writable:

```javascript
function crossPlatformFunctionUsingNodes(node) {
  'use strict';
  // 👆 this forces errors when assigning non-writable properties

  let writable = false;
  try {
    node.innerHTML = 'Some value';
    writable = true;
  } catch {
    writable = false;
  }
  if (writable) {
    // They can write to the node
  }

  // Alternatively, we can use a more sophisticated function to do that detection,
  // like the one defined in https://www.npmjs.com/package/@stdlib/assert-is-writable-property
  const isWritableProperty = require('@stdlib/assert-is-writable-property');
  if (isWritableProperty(node, 'innerHTML')) {
    // Some logic
  }
}
```

Type checkers like TypeScript and Flow should also help with these, reducing the amount of runtime checks necessary to safely access these APIs. Some APIs return nodes, so doing runtime checks to refine them as elements is required, but other APIs can be safely accessed if the type definitions already account for the differences between platforms.

For example, the definition for the type of ref returned by `div` would be an `Element` instance, whereas the type of ref returned by `View` would be a `ReadOnlyElement`. Accessing the intersection between those types should be safe in both environments. Modifying `innerHTML` on `Element` would be safe when type checking code targeting the Web, but would fail on `ReadOnlyElement` when targeting React Native.

In general, **the `instanceof` operator should be avoided**. It would not work in React Native where `Node` and `Element` are not defined directly (but as `ReadOnlyNode` and `ReadOnlyElement`), which would platform-specific refinements. Instead, users should use feature detection directly, or check for the `node.nodeType` value for a better way to implement feature inference.

## Drawbacks

### General access to the host tree

The main drawback of this proposal is the potential for misuse or abuse in the access to the host tree. In React DOM, accessing the DOM directly is considered an escape hatch that should be avoided whenever possible (e.g.: avoiding refs for anything that can be done declaratively). That would continue applying in React Native.

Something that mitigates this risk in React Native specifically is the fact that the framework defines all the objects in the view hierarchy. That means we can easily add logic to their methods to detect when they are potentially being misused (e.g.: invoking them during render).

### Larger surface area and strict semantics

This proposal would expose additional APIs in userland, together with some expectations about the underlying model that React Native uses to render UI and execute JavaScript and native code. This could prevent or complicate future use cases and optimizations in the framework, if they required changes in the new semantics we are exposing in this proposal.

The main assumptions this proposal makes about semantics, which could affect future changes to Fabric and other parts of the framework, would be:

1. Having a host tree representing the structure and state of the UI.
2. Having fast access to the tree (no major synchronization costs).
3. Including layout information in the tree.
4. Having updates to the tree that are synchronized with the execution of JS and the React lifecycle, so they are observable in the points described by the proposal (after commits in React and between JS tasks).

All of these are very unlikely to change, considering they are also assumptions made by the React reconciler in its interaction with the host platform.

### DOM subset

A question that comes up often in React Native is whether new functionality should be provided using APIs aligned with Web standards, or else as React Native specific APIs.

It seems there is some consensus in the community about using Web APIs when:

1. They solve the same use cases.
2. The APIs provided by the Web are suitable for the React Native model.
3. The APIs are mostly compatible across platforms.

This proposal would provide a subset of the DOM traversal APIs that would be compatible with the Web, which would be aligned with the consensus. This proposal also contemplates providing useful warnings when using unimplemented parts of the specification that are commonly used (like assigning values to [`innerHTML`](https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML) or [`innerText`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/innerText)).

## Alternatives

An alternative solution for accessing the host tree in React Native would be creating React Native specific APIs that do not match the DOM. That would be a worse solution because we would have to expose a similar API surface area and would still need to provide semantic guarantees, without the benefit of allowing cross-platform code written in JavaScript. This would also be against the general direction of the framework where we are looking to converge on the APIs used across platforms.

Another alternative would be to continue disallowing access to the host tree. Many of the use cases this unblocks could be implemented in native, but that has 2 big problems:

1. Implementing this in native seems to have less implications for APIs changes, but that is only because most of the native APIs that React Native exposes are currently undocumented.
2. This requires users to implement platform-specific code, which prevents having cross-platform libraries that support new platforms. This is against the Many-Platforms vision for React Native.

## Adoption strategy

The proposed APIs would only be available in the new React Native architecture, so its rollout and documentation should be done after it (or in parallel).

Additionally, we would gate these APIs behind a feature flag until they are stable. This is to avoid having unstable APIs exposed direclty or having non-standard names for them (like `unstable_getBoundingClientRect`) that would have to be migrated after.

We should document these APIs in the [React Native website](https://reactnative.dev/):

1. Adding a new guide about imperatively accessing the host tree.
2. Creating a new dropdown section for globals in the APIs section (with reference documentation for classes like `ReadOnlyNode`, `ReadOnlyElement` and `ReactNativeElement`).
3. Adding a reference to the API provided by refs in native components (link to the definition of `ReactNativeElement`).

In order to promote these new capabilities, we can do things like:

1. Creating a blog post about the new APIs.
2. Referencing these APIs as part of the rollout of the new architecture, as one of the use cases that it unblocks.
3. Creating demos showing how these APIs can be used in React DOM and React Native without changes.
4. Promoting this in our social channels, like Twitter and Discord.

Additionally, we can consider adding React Native to the [browser compatibility tables in MDN](https://developer.mozilla.org/en-US/docs/Web/API/Element#browser_compatibility).

## How we teach this

We would leverage the familiarity of users with DOM APIs to teach this. In the case of semantic differences, we would try to signal those at runtime with links to the React Native documentation for easy discoverability.

## Unresolved questions

* Definition of [`tagName`](https://developer.mozilla.org/en-US/docs/Web/API/Element/tagName) (and [`nodeName`](https://developer.mozilla.org/en-US/docs/Web/API/Node/nodeName)) for built-in components in React Native and custom components defined in userland.

## Future extensions

This is a **non-exhaustive** list of **possible** future extensions to the framework related to this proposal:

* Synchronized access to the host tree with synchronous events and APIs like [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame).
* Support for APIs like [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver), [`MutationObserver`](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) and [`ResizeObserver`](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver).
* Support for methods to access attributes (e.g.: [`getAttribute()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAttribute), [`getAttributeNames()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAttributeNames), etc.).
* Support for `ReadOnlyElement` subclasses in built-in and custom components (e.g.: `ReadOnlyImageElement` with methods to access information about the image).
* Support for imperatively attaching event listeners (implementing the [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) API).
* Support for a subset of the [`Document`](https://developer.mozilla.org/en-US/docs/Web/API/Document) API with properties like [`activeElement`](https://developer.mozilla.org/en-US/docs/Web/API/Document/activeElement), [`visibilityState`](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilityState),  [`elementFromPoint()`](https://developer.mozilla.org/en-US/docs/Web/API/Document/elementFromPoint), etc.

## Changelog

|Date	|Author	|Change	|
|---	|---	|---	|
|2023-03-01	| [Rubén Norte](https://github.com/rubennorte)	|Initial version published	|
|2023-03-02	| [Rubén Norte](https://github.com/rubennorte)	|Fixed information about Reanimated	|
|2023-03-10	| [Rubén Norte](https://github.com/rubennorte)	| Added rollout strategy with feature flag	|
