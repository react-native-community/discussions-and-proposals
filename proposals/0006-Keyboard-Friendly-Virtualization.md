---
title: "RFC0006: Keyboard-friendly List Virtualization Behavior"
author: Nick Gerleman
date: February 2021
---

# RFC0006: Keyboard-friendly List Virtualization Behavior

## Summary

VirtualizedList virtualizes away items based on proximity to the lists viewport,
without consideration for other features such as focus or selection. This
strategy limits the ability to correctly implement some common keyboard
interactions. This especially impacts desktop platforms (react-native-web,
react-native-windows, react-native-macos), but also impacts the tablet with
keyboard form-factor on iPadOS/Android.

This proposal outlines these issues, proposing a set of behavior changes and new
APIs to address them. It does not address concrete implementation.

## Basic example

```jsx
<FlatList
  {...otherProps}
  keepAreasRealized={{
    focused: false,
    home: [true, {windowSize: 1}],
  }}
/>
```

## Motivation

### Keyboarding issues with VirtualizedList (and FlatList/SectionList)

Keyboard users expect to be able to navigate between between views inside of a
list via tab-loop or arrow keys. A component with focus may move out of
visibility by panning or scrolling a parent surface. The focused component may
be virtualized away after leaving visibility.

Removing components in the focus chain breaks some common keyboard scenarios:
1. Moving to a previous/next selected item via arrow-key
1. Tabbing focus to a user-definable next component
1. Imperatively moving focus to a user-defined component

## Detailed design

### Realization windows

Focus may be synchronously moved by the native UI framework. This forces us to
prevent virtualization (keep realized) any components which focus may be moved
to. We may see additional navigation after the initial focus change, unblocked
on the rendering of new components. This necessitates a chain of additional
realized components.

Common keyboarding scenarios should "just work" without having to change
settings. Three of these scenarios include:

1. Navigating from one child component of a rendered item to another
1. Arrowing between focused items
1. Shifting to first/last focused item via home/end keys

`VirtualizedList` has enough internal knowledge to transparently support these
scenarios. This can be accomplished by adding three **"realization windows"**:
1. **"focused"** Realize cells around a cell with active focus. Can be
tracked by listening to bubbling onFocus in the default `CellRendererComponent`.
1. **"home"** The area at the beginning of the list (for home key).
1. **"end"** The area at the end of the list (for end key).

The above scenarios are not exhaustive. E.g. we may want to keep a *selected*
item realized that is not *focused*. These scenarios may require additional APIs
allowing external users greater control over realization behavior. These could
potentially be exposed in the future as dynamic/user-controllable realization
windows, or separate high-level APIs.

### Endpoint-specific default behaviors

VirtualizedList should be keyboard-friendly by default on desktop platforms as
they are oriented towards usage of a physical keyboard. This additional
realization is not free however, increasing memory usage to retain additional
cells. This tradeoff may not make sense for mobile.

This leads to desired behavior of:
- **Keep FlatList focus-safe on desktop:** Common realization windows for
needed for keyboard navigation are enabled by default.
- **Keep FlatList memory usage steady on mobile:** New realization windows
are opt-in where not pay to play.

The full matrix of behaviors is described in the *API Reference* below. In
general, **desktop users should realize **

Endpoint detection is generally performed via `Platform` APIs, such as
`Platform.isTV()` or `Platform.isPad()`. This change would add a
`Platform.isDesktop()` endpoint query to allow desktop-specific behavior.

### New VirtualizedList props

Properties should be added to VirtualizedList (and FlatList/SectionList) to
allow optimizing for app-specific needs. This can be exposed as a
`RealizationWindowConfig`, describing the size and enablement of predefined
realization windows. This API could be expanded in the future to allow
dynamically controlled realization behaviors.


### API reference

```tsx
/**
 * Allows enabling or disabling an area for realization, optionally overriding
 * its window size.
 */
export type RealizationWindow =
  | boolean
  | [boolean, {windowSize?: number}];

/**
 * Determines what areas of the list to keep realized. Enabling realization may
 * be required for correct behavior when focus is out of viewport, but may also
 * increase memory usage.
 */
export type RealizationWindowConfig = {
  /**
   * Keeps the area around the currently focused item realized.
   * - Enabled by default on desktop/tv (where focus-loop is important)
   * - Enabled by default on mobile, where this can only use more memory when
   * physical-keyboard is used.
   *
   * Requires the default (CellRenderComponent)[
   * https://reactnative.dev/docs/virtualizedlist#cellrenderercomponent] to be
   * used
   *
   * Defaults to the (VirtualizedList windowSize)[
   * https://reactnative.dev/docs/virtualizedlist#windowsize]
   */
  focused?: RealizationWindow;

  /**
   * Keeps the beginning area of the list realized.
   * - Enabled by default on desktop (for home-key navigation)
   * - Disabled by default on mobile/TV
   *
   * Defaults to **half** the (VirtualizedList windowSize)[
   * https://reactnative.dev/docs/virtualizedlist#windowsize]
   */
  home?: RealizationWindow;

  /**
   * Keeps the end area of the list realized.
   * - Enabled by default on desktop (for end-key navigation)
   * - Disabled by default on mobile/TV
   *
   * Defaults to **half** the (VirtualizedList windowSize)[
   * https://reactnative.dev/docs/virtualizedlist#windowsize]
   */
  end?: RealizationWindow;

  // Potential to add additional props for different (or dynamic) realization
  // areas
}

/**
 * VirtualizedList (and FlatList/SectionList) component properties
 */
export type VirtualizedListProps = {
  ...

  /**
   * Enables realization of different areas of the VirtualizedList
   */
  keepAreasRealized?: RealizationWindowConfig;
}
```

## Drawbacks

Apart from implementation complexity and risk, the greatest drawback to
additional realization is memory consumption. This motivates configurability to
mitigate impact.

## Alternatives

The main alternative to improving keyboarding support of existing components on
desktop platforms would be to add additional virtualization components unique to
them.
