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
- Native code resolves the conditions, so a color scheme change does not require React re-render.
- Implementation, as 3 stacked PRs:
  1. Core: conditional values, resolution and updates ([#60124](https://github.com/react/react-native/pull/60124))
  2. iOS and Android set the color scheme ([#60125](https://github.com/react/react-native/pull/60125))
  3. Flow/TS types and an RNTester example ([#60126](https://github.com/react/react-native/pull/60126))

## Basic example

```js
import { StyleSheet, View } from "react-native";

function Card() {
  return <View style={styles.card} />;
}

const styles = StyleSheet.create({
  card: {
    // A single property can carry a default plus media-query branches.
    backgroundColor: {
      default: "#fff",
      "@media (prefers-color-scheme: dark)": "#111",
    },
    borderColor: {
      default: "#ddd",
      "@media (prefers-color-scheme: dark)": "#444",
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

A conditional value is a plain object with a `default` key and one or more `'@media …'` keys:

```js
backgroundColor: { default: "white", '@media (prefers-color-scheme: dark)': "black" }
```

- An object value with a `default` key or an `@media` key is a conditional value. Everything else in a style object is authored exactly as before (plain object values like `shadowOffset` are untouched), so it is backward compatible.
- `default` is required. Without it, the prop gets no value (`null`) and a development error is logged. `default: null` means "no value" on purpose.
- Conditions on a property are evaluated in source order and the last matching one wins. When none match, `default` applies.
- A query that is not supported is ignored, with a development error.

The conditions live per property rather than as nested `@media` blocks grouping whole style objects (`'@media (…)': { backgroundColor: 'black' }`). The [React DOM for Native RFC (#496)](https://github.com/react-native-community/discussions-and-proposals/pull/496) weighed both forms and favored the per‑property one, because it makes composition deterministic: since a condition lives on the property, any later style in a `style` array overrides that property and all of its conditions. This fits RN's atomic styling model, where `flatten` resolves arrays per key.

Supported media features in the current scope:

| Feature                | Values            | Evaluated against                                       |
| ---------------------- | ----------------- | ------------------------------------------------------- |
| `prefers-color-scheme` | `light` \| `dark` | The value of `useColorScheme()`, with the app overrides |

`prefers-contrast`, `(min|max)-(width|height)` and `orientation` can be built on the same approach (see [Future media features](#future-media-features)).

Conditional values work for the props in the core style table. 5 style props are not supported yet, because components read and convert them in JavaScript: `objectFit`, `resizeMode`, `tintColor` (`Image`), and `userSelect`, `verticalAlign` (`Text`). See [Unresolved questions](#unresolved-questions).

### How it works

Conditional styles are authored in JavaScript, but resolved on the native side. Changing the color scheme re-resolves them natively, without a React re-render.

**1. Payload (JS).** When React builds the props payload, a conditional value of a style prop is sent under its own prop, and the payload gets a flag:

```js
// authored
{ opacity: { default: 1, '@media (prefers-color-scheme: dark)': 0.5 } }

// sent to native
{
  opacity: {
    $conditionalValue: {
      default: 1,
      conditions: [{ query: { colorScheme: 'dark' }, value: 0.5 }],
    },
  },
  $hasConditionalValues: true,
}
```

- Each value goes through the prop's normal processor (e.g. `processColor`), so once matched it behaves like any other value.
- A later plain value for the same prop (in a `style` array, or in a later update) replaces the condition:

  ```js
  const styles = StyleSheet.create({
    card: {
      backgroundColor: { default: 'white', '@media (prefers-color-scheme: dark)': 'black' },
    },
    selected: { backgroundColor: 'blue' },
  });

  <View style={[styles.card, styles.selected]} />; // blue in light and dark mode
  ```

**2. Resolution (native, when props are parsed).** `ConcreteComponentDescriptor::cloneProps` resolves conditional values **before** it parses the props. The parser only sees plain values, so every parse path works without changes: the classic parser, the iterator setter, and the Android raw props (Props 1.0, 1.5 and 2.0). The props keep a small record:

- the conditions of each prop,
- the media features that the conditions read (`dependencies`),
- the media features that the values were resolved for (`resolvedFor`).

A payload from React without the flag is skipped after one property lookup.

**3. Updates (native, during commit).** `UIManager::setMediaFeatures` saves the new color scheme in a store (one per React Native instance) and commits each surface. In every commit, `ShadowTree::tryCommit` resolves again the views that were resolved for an older color scheme, before the commit hooks and layout. It runs in every commit because the native commit does not update React's copy of the views, so React's next commit can bring back the old values.

**Optimizations.**

- **In apps without conditional values, the pass stops at the root.**
- **Subtrees without conditional values are skipped** (`ChildrenHaveConditionalValues` trait).
- **Only views that read the changed feature are resolved again.** Each view stores the features it reads (`dependencies`), so a future `width` change will not touch views that only read the color scheme.
- **Only those views and their ancestors are cloned.** All other nodes are shared with the previous tree. If nothing is outdated, the tree does not change.

### iOS

`RCTSurfacePresenter` sets the initial color scheme in the store before it creates the scheduler, so the first render uses it. It observes `RCTUserInterfaceStyleDidChangeNotification` and calls `setMediaFeatures`. The value comes from `RCTColorSchemePreference`, the function that the `Appearance` module uses. So conditional styles match `useColorScheme()`, also with the app overrides (for example `RCTOverrideAppearancePreference`). The function moves from React-CoreModules to React-Core, so Fabric can call it. `Appearance.setColorScheme()` works, because it changes the window style.

### Android

The binding sets the initial color scheme from the configuration's `uiMode` before it creates the scheduler. After that, `AppearanceModule` sends each new color scheme to `FabricUIManager.setMediaFeatures`. So conditional styles match `useColorScheme()`, also with an `OverrideColorScheme`, which the module sends when it is created. `Appearance.setColorScheme()` works, because it changes the activity configuration.

### Future media features

The same approach can support:

- `orientation`
- `prefers-contrast`
- `(min|max)-(width|height)`
- and more

## Drawbacks

- **Apps without conditional values:** only one property lookup per props update (about 0.05 µs) and a check at the root in each commit. A whole commit of 1000 views, check included, took about 0.35 µs (Mac, Release build). The lookup creates the flag name as a JS string each time; a cached `jsi::PropNameID` can improve this.
- **With conditional values:** about +3.5 µs to create a view whose payload has a conditional value (+2 µs for an update), and about +5 µs per commit for 1000 conditional views. Most of the extra cost per payload comes from converting the whole payload to `folly::dynamic`; converting only the conditional values can improve this.
- Components that read a style value in JavaScript get the conditional object, not the resolved value (for example `TouchableOpacity` reading `opacity`).
- **Types:** `StyleProp<T>` accepts conditional values, so a value read after `StyleSheet.flatten(style)` can be a `ConditionalStyleValue`. TypeScript code that reads one style value must handle the object or cast. Passing flattened styles on still compiles.

## Alternatives

**JS hooks (`useColorScheme`, `useWindowDimensions`).** Works today, but every change re‑renders React and runs on the JS thread; `useWindowDimensions` updates asynchronously, so resizing large trees can visibly lag. This proposal moves resolution off the JS thread.

**[react-native-unistyles](https://github.com/jpudysz/react-native-unistyles).** Also updates styles without a React render: its C++ core commits new styles to the Fabric shadow tree. But its stylesheets are JavaScript functions of the theme, so a change calls them again on the JS thread. This proposal resolves the conditions in the native commit, with no JavaScript and no Babel setup. Unistyles offers much more (variants, multiple themes).

**[NativeWind](https://github.com/nativewind/nativewind).** Tailwind for React Native. Classes compile to style objects at build time, but the rules for the color scheme or dimensions apply at runtime in JS, and the affected components render again. So it is closer to the JS hooks above.

**[One](https://github.com/onejs/one).** Its `@vxrn/color-scheme` package stores a light, dark or system setting. On native, it uses `Appearance` and a React hook, so a change renders React again. On web, it sets a `t_dark` or `t_light` class on `<html>`, so CSS applies the theme. Its setter calls `Appearance.setColorScheme()`, so with this proposal, conditional styles would follow it natively.

There are a few more libraries; most use the hook/listener API model and require a React re-render.

If React Native implements native media query support, the above libraries could use it under the hood as a primitive.

## Adoption strategy

- **Opt‑in.** Only object values with a `default` or `@media` key are treated as conditional. Existing styles are untouched.
- **No feature flag.** Without conditional values, the feature does no work except the small checks listed in [Drawbacks](#drawbacks). A flag can be added if a staged rollout is wanted.
- **New Architecture only.** The payload step lives in the Fabric attribute payload.
- **Typing.** Flow and TypeScript types are included, with the type change described in [Drawbacks](#drawbacks).
- **Incremental feature rollout.** `prefers-color-scheme` first. `prefers-contrast`, `min/max-(width|height)` and `orientation` can follow on the same primitive without changing the authoring shape.

## How we teach this

A **"Conditional style values"** section in the `StyleSheet` docs: a property's value can be `{ default, '@media (…)': value }`, with the supported‑features table, the rule that `default` is required, and the props that do not support conditional values yet.

## Unresolved questions

**Props that components convert in JavaScript** (`objectFit`, `resizeMode`, `tintColor`, `userSelect`, `verticalAlign`). Moving these conversions to native props parsing would make conditional values work for them. This could be a follow-up.
