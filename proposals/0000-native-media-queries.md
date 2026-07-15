---
title: Native media queries in StyleSheet
author:
  - Nishan Bende
date: 2026-07-09
---

# RFC0000: Media query support in StyleSheet

## Summary

- This proposal adds media query support to React Native's `StyleSheet`.
- It uses StyleX's [syntax](https://stylexjs.com/docs/learn/styling-ui/defining-styles#media-queries-and-other--rules) for conditional styles. The same syntax [RSD](https://react.github.io/react-strict-dom/learn/styles#media-queries) uses. It works well with RN's atomic styling model.
- Draft PR - [facebook/react-native#57439](https://github.com/facebook/react-native/pull/57439)

## Basic example

```js
import { StyleSheet, View } from "react-native";

function Card() {
  return <View style={styles.card} />;
}

const styles = StyleSheet.create({
  card: {
    // A single property can carry a default plus media-query branches.
    padding: {
      default: 12,
      "@media (orientation: portrait)": 24,
      "@media (orientation: landscape)": 36,
    },
    backgroundColor: {
      default: "#fff",
      "@media (prefers-color-scheme: dark)": "#111",
    },
  },
});
```

## Motivation

- Most apps need dark/light mode styling. Currently it requires the `useColorScheme` hook, which triggers React re-renders that keep the JS thread busy and can sometimes cause the whole app to re-render on a color mode change. This has also appeared as a pain point in the State of React Native 2025 survey - https://results.2025.stateofreactnative.com/en-US/styling/
- Many apps need the capability to write dimension/orientation dependent styling. Currently it requires `useWindowDimensions` hook. The hook triggers asynchronously, driven by an event from native which can make UI appear laggy when resizing large UI.
- We want to bring StyleSheet's capabilities closer to web while preserving its atomic styling philosophy.
- Making responsive styles easier in foldable/different devices.

## Detailed design

### API

The API is borrowed from StyleX. A property's value can be a conditional style value, i.e. a value that changes when a condition is satisfied.

A conditional value is a plain object with a required `default` key and one or more `'@media …'` keys:

```js
backgroundColor: { default: "white", '@media (prefers-color-scheme: dark)': "black" }
```

- `default` is required only when the value is an object and one of the keys starts with `@media`, everything else in a style object is authored exactly as before (plain object values like `shadowOffset` are untouched). So it is fully backward compatible.
- Conditions on a property are evaluated in source order and the last matching one wins. When none match, `default` applies.
- A query can combine features with `and`: `'@media (orientation: landscape) and (prefers-color-scheme: dark)'`.

The conditions live per property rather than as nested `@media` blocks grouping whole style objects (`'@media (…)': { backgroundColor: 'black' }`). The [React DOM for Native RFC (#496)](https://github.com/react-native-community/discussions-and-proposals/pull/496) weighed both forms and favored the per‑property one, because it makes composition deterministic: since a condition lives on the property, any later style in a `style` array overrides that property and all of its conditions. This fits RN's atomic styling model, where `flatten` resolves arrays per key.

Supported media features in the draft PR:

| Feature                | Values                    | Evaluated against        |
| ---------------------- | ------------------------- | ------------------------ |
| `orientation`          | `landscape` \| `portrait` | surface width and height |
| `prefers-color-scheme` | `light` \| `dark`         | OS / app color scheme    |

`((min|max)-(width|height))` media queries can be easily built on top of the approach discussed in this RFC.

### How it works

Conditional styles are authored and compiled in JavaScript, but resolved on the native side. Changing the color scheme or rotating the device re-resolves them natively during the shadow tree commit, without a React re-render.

**1. Compilation (JS).** The style-processing path compiles each conditional value into two parts: the inline `default` (parsed and processed exactly like a normal value) plus a single `styleConditions` prop that collects the parsed conditions, keyed by property:

```js
// authored
{ width: { default: 120, '@media (orientation: landscape)': 300 } }

// compiled
{ width: 120, styleConditions: { width: [{ query: { orientation: 'landscape' }, value: 300 }] } }
```

- Each condition value flows through the property's normal processor (e.g. `processColor`), so once matched it behaves like any other value.

**2. Resolution (native, during commit).** The `styleConditions` prop is added to the node's props. When the shadow tree commits (in `ShadowTree`, right before layout, so resolved values participate in it), it walks the tree and, for each node carrying conditions, evaluates its queries against the current environment and patches the matched value onto the props (or restores the unpatched props when nothing matches).

- Patched props keep a pointer to their unpatched base, and JS prop updates are parsed over the base, so native patches are not visible to JS and always revert cleanly.
- A `HasStyleConditionsInSubtree` trait lets the walk skip subtrees with no conditional styles, so cost is proportional to the paths reaching conditional nodes, not the whole tree.

**3. The media query resolution is derived per‑surface from the shadow tree's root.**

- **Orientation** is derived from the surface's own viewport (the root's maximum layout size: `landscape` when width > height). It needs no platform plumbing, any rotation/resize already produces a layout commit (`constraintLayout` function), which re‑runs the resolution.
- **Color scheme** is stored per surface (a field on each root shadow node's props) but is an app-wide value, matching `useColorScheme`/`Appearance`. A commit is triggered whenever it is updated. It currently listens to app-wide color scheme changes and stores the value on each surface's root (this could be made per-surface in the future).

### iOS

`RCTSurfacePresenter` observes the `RCTUserInterfaceStyleDidChangeNotification` and pushes the effective color scheme to each surface. The initial color scheme is set on surface start. The scheme is resolved from window trait collections, so `Appearance.setColorScheme()` should be honored and the result matches `useColorScheme()`. Orientation change uses the existing layout commit which triggers the same resolution.

### Android

The draft PR gates the feature on Android until Props 2.0 (typed prop diffing) lands, which will make it work the same way as iOS. Android will require minimal work (setting the color scheme) once Props 2.0 is enabled. Until then, undoing a resolution patch when the environment switches back requires storing the full accumulated props, instead of just the keys each JS update sends. We can make this work on Props 1.0, but the feature flags in the Android props path make it difficult to review and it becomes throwaway work the moment 2.0 lands 😅

## Drawbacks

- Resolution runs on every commit while conditions are in use (optimised by the `HasStyleConditionsInSubtree` trait, but non‑zero).
- Adds runtime work in JS StyleSheet i.e. iteration on keys in style prop to process styles in JS. This can be optimised in future via babel plugin, RN's runtime flattening makes it tricky though. It is currently optimised with a weak map cache so it should not be too bad if style object stays the same by reference.

## Alternatives

**JS hooks (`useColorScheme`, `useWindowDimensions`).** Works today, but every change re‑renders React and runs on the JS thread; `useWindowDimensions` updates asynchronously, so resizing large trees can visibly lag. This proposal moves resolution off the JS thread.

**[react-native-unistyles](https://github.com/jpudysz/react-native-unistyles).** A library pursuing the same goal of updating styles without a React re‑render. Its C++ core links each component's shadow node to its computed styles and commits updates directly onto the Fabric shadow tree, so a theme or dimension change updates without rendering React, mechanically close to this proposal. The difference is that Unistyles styles are authored in JavaScript (objects that read the theme, or functions), so when the environment changes it re-resolves them by executing that JavaScript on the JS thread. This proposal instead compiles the conditions ahead of time and resolves them entirely inside the native commit, with no JavaScript at resolution time and no Babel transform setup required. Unistyles also offers a lot more (variants, multiple themes, etc.).

**[NativeWind](https://github.com/nativewind/nativewind).** Tailwind for React Native. Classes are statically compiled to style objects at build time, but which rules apply for the current color scheme, dimensions, or pseudo‑state is resolved at runtime in JS: platform signals arrive via `Appearance`/`Dimensions` listeners and propagate through an observable that re‑renders each affected component (scoped to those components, but on the JS thread). So for the conditions this proposal covers, it is closer to the JS‑hooks alternative above.

There are a few more libraries; most use the hook/listener API model and require a React re-render.

If React Native implements native media query support, the above libraries could use it under the hood as a primitive.

## Adoption strategy

- **Opt‑in and fully backward compatible.** Only a value that is an object with at least one `@media` key is treated as conditional, existing styles are untouched. No breaking change.
- **Feature flag.** For rollout, the feature should ship behind a `ReactNativeFeatureFlags` flag (e.g. `enableMediaQueryStyles`) gating its two entry points: the JS compilation step in the `ReactNativeAttributePayload` and the native resolution during commit. The draft PR keeps the feature ungated to make it easy to build and test.
- **New Architecture only.** The compile step lives in the Fabric attribute payload.
- **Typing.** The draft PR ships without type changes to `StyleSheet`. A follow‑up will add the type support.
- **Incremental feature rollout.** `prefers-color-scheme` + `orientation` first, `min/max-(width|height)` can follow on the same primitive without changing the authoring shape.

## How we teach this

A **"Conditional style values"** section in the `StyleSheet` docs: a property's value can be `{ default, '@media (…)': value }`, with the supported‑features table and the rule that `default` is required.

## Unresolved questions

- **Android support** Do we have plans when we're rolling out Props 2.0 on android?
