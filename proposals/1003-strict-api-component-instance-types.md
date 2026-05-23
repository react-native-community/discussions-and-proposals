---
title: Add dedicated Instance types for built-in component refs (JS API)
author:
  - Alex Hunt
date: 2026-05-23
---

# RFC1003: Add dedicated Instance types for built-in component refs (JS API)

## Summary

Add a publicly exported `*Instance` type for every built-in React Native component (e.g. `ViewInstance`, `TextInputInstance`, `ScrollViewInstance`), providing a canonical way to type refs. This is a missing piece of our API, historically hidden by the use of class declarations in the hand-written TypeScript types.

## Basic example

```tsx
import { useRef } from "react";
import type { ViewInstance, TextInputInstance } from "react-native";

function MyComponent() {
  const viewRef = useRef<ViewInstance>(null);
  const inputRef = useRef<TextInputInstance>(null);

  return (
    <>
      <View ref={viewRef} />
      <TextInput ref={inputRef} />
    </>
  );
}
```

These types are also valid for `Animated` variants — the `Animated` wrapper is transparent for refs:

```tsx
const viewRef = useRef<ViewInstance>(null);

<Animated.View ref={viewRef} />;
```

## Motivation

Typing refs for built-in components is harder than it should be, and it's about to get harder.

- **No canonical pattern**: There is currently no canonical way to type a ref to a built-in React Native component.
- **Three competing approaches**: Developers face three different patterns depending on the language, the types version, and which documentation they read. None is satisfying on its own.
- **Migration blocker**: The upcoming [Strict TypeScript API](https://reactnative.dev/docs/strict-typescript-api) migration breaks the most common pattern (`useRef<View>`) across every app that uses it. Without a clean replacement, this is the single highest-friction change in the migration.

### The class component legacy

React Native's hand-written TypeScript definitions (the `types/` directory shipped with `react-native`) declare all built-in components as classes. This means `View`, `Text`, `ScrollView`, etc. are simultaneously values (the component) and types (the class instance):

```ts
// Works today with hand-written .d.ts types — View is a class
const viewRef = useRef<View>(null);
```

This shorthand is widely used in the ecosystem and works because TypeScript's class declarations produce both a value and an instance type.

### The Strict TypeScript API migration

React Native is migrating to auto-generated TypeScript types derived from the Flow source ([Strict TypeScript API](https://reactnative.dev/docs/strict-typescript-api)). In these generated types, built-in components are typed as functions, not classes. This breaks the `useRef<View>` pattern — `View` as a type now refers to the function itself, not the native element instance:

```ts
// With generated types — View is a function, not a class
const viewRef = useRef<View>(null);
//                      ^^^^ This is now typeof View (the function),
//                           not the native element instance
```

**Why it matters**: This is the single most disruptive change in the Strict TypeScript API migration, and a [documented known issue](https://reactnative.dev/docs/strict-typescript-api#some-core-components-are-now-function-components-instead-of-class-components). Without a clean solution, every ref annotation in every React Native app needs a non-obvious fix.

### The current workarounds

Today, developers can work around this with utility types:

```ts
// Flow
const viewRef = useRef<React.ElementRef<typeof View>>(null);

// TypeScript
const viewRef = useRef<React.ComponentRef<typeof View>>(null);
```

This is correct but verbose and unfamiliar. It also differs between Flow ([`React.ElementRef`](https://flow.org/en/docs/react/types/#toc-react-elementref)) and TypeScript (`React.ComponentRef`), adding to the confusion.

Alternatively, developers can use the base `HostInstance` type:

```ts
import type { HostInstance } from "react-native";

const viewRef = useRef<HostInstance>(null);
```

**Key problem**: This compiles but is too broad — it loses component-specific imperative methods. For example, `TextInput` exposes `clear()`, `isFocused()`, and `setSelection()`, and `ScrollView` exposes `scrollTo()`, `scrollToEnd()`, and `flashScrollIndicators()`. Using `HostInstance` for these refs silently discards access to those methods.

### The problem, in short

| Pattern                                         | Works in Flow      | Works in TS (manual) | Works in TS (generated) | Preserves methods (functional correctness) |
| ----------------------------------------------- | ------------------ | -------------------- | ----------------------- | ------------------------------------------ |
| `useRef<View>(null)`                            | No                 | Yes (class)          | No (function)           | N/A                                        |
| `useRef<React.ComponentRef<typeof View>>(null)` | Yes (`ElementRef`) | Yes                  | Yes                     | Yes                                        |
| `useRef<HostInstance>(null)`                    | Yes                | Yes                  | Yes                     | No                                         |

### Why this solution

**Bottom line**: None of these workarounds is a good default recommendation. `*Instance` types solve this by giving developers a concise, predictable target for each component.

- **Minimal migration**: `useRef<View>` becomes `useRef<ViewInstance>` — preserving the mental model while reducing the net difficulty of the Strict TypeScript API migration.
- **Web parity**: On the web, every HTML element has a dedicated type (`HTMLDivElement`, `HTMLInputElement`, `HTMLImageElement`) that developers use for refs without hesitation. React Native has never had this — `*Instance` types fill the same role.
- **Future-proofing**: Several built-in components (`ImageBackground`, `KeyboardAvoidingView`, `RefreshControl`, etc.) are still classes today. When these are eventually converted to function components, code using `*Instance` types will continue to work — whereas `useRef<RefreshControl>` would silently break, exactly as `useRef<View>` does today.

### Relation to other efforts

This proposal is complementary to [Removing deep imports from react-native](https://github.com/react-native-community/discussions-and-proposals/pull/894), which consolidates the public API surface of the `react-native` package. Instance types are part of that public surface — they should be top-level exports, not deep imports from private paths.

## Detailed design

**Draft implementation**: [#56953](https://github.com/facebook/react-native/pull/56953).

### New exports

Add a `*Instance` type export for every built-in component that supports refs. Each type is co-located with its component source and re-exported from the package entry point:

```ts
import type {
  ViewInstance,
  TextInstance,
  TextInputInstance,
  ScrollViewInstance,
  ImageInstance,
  PressableInstance,
  FlatListInstance,
  // ... etc.
} from "react-native";
```

**💡 Key fix**: These are top-level, public exports — not deep imports from private paths. This eliminates the need for workarounds like `import type {PublicScrollViewInstance} from 'react-native/Libraries/Components/ScrollView/ScrollView'` that some libraries have resorted to.

### Type definitions

Most components' instance types alias `HostInstance` (which is `ReactNativeElement` — the base native element class with methods like `measure()`, `focus()`, `blur()`):

```
ViewInstance        = HostInstance
TextInstance        = HostInstance
ImageInstance       = HostInstance
PressableInstance   = HostInstance
SwitchInstance      = HostInstance
// ... most components
```

Components with additional imperative methods extend the base:

```
TextInputInstance extends ReactNativeElement {
  clear(): void;
  isFocused(): boolean;
  getNativeRef(): ReactNativeElement | null;
  setSelection(start: number, end: number): void;
}

ScrollViewInstance extends HostInstance, ScrollViewImperativeMethods {
  scrollTo(...): void;
  scrollToEnd(...): void;
  flashScrollIndicators(): void;
  getScrollResponder(): ScrollResponderType;
  // ... etc.
}
```

**💡 Key fix**: These specialised instance types are the cases where `HostInstance` silently loses type safety today. Dedicated `*Instance` types ensure developers get autocomplete and type checking for component-specific imperative methods.

### Naming convention

- Pattern: `{ComponentName}Instance` (e.g. `ViewInstance`, `TextInputInstance`).
- This aligns with React Native's existing naming (`HostInstance`, `TextInputInstance` which already exists internally) and the `Instance` terminology used in React's own types (`React.ComponentRef`, `React.ElementRef`).
- We intentionally diverge from the web convention (`HTMLDivElement`, `HTMLInputElement`) to maintain a layer of indirection for React Native ref types where an "instance" may not be 1:1 with a native element.

### Animated components

`Animated.View`, `Animated.Text`, etc. accept the same ref types as their non-animated counterparts. The `Animated` wrapper created by `createAnimatedComponent` forwards the ref to the underlying native component. No separate `Animated.*Instance` types are needed:

```tsx
// ViewInstance works for both
<View ref={viewRef} />
<Animated.View ref={viewRef} />
```

**💡 Key fix**: This replaces the removed `Animated.LegacyRef` type from the strict types. Code using `ref={ref as React.Ref<Animated.LegacyRef<View>>}` can simply use `ref={ref}` with a `ViewInstance`-typed ref.

### Full component table

| Component                 | Instance Type                     | Type Value                                     |
| ------------------------- | --------------------------------- | ---------------------------------------------- |
| `ActivityIndicator`       | `ActivityIndicatorInstance`       | `HostInstance`                                 |
| `Button`                  | `ButtonInstance`                  | Platform-dependent                             |
| `DrawerLayoutAndroid`     | `DrawerLayoutAndroidInstance`     | `DrawerLayoutAndroidMethods`                   |
| `FlatList`                | `FlatListInstance`                | `FlatList<>`                                   |
| `Image`                   | `ImageInstance`                   | `HostInstance`                                 |
| `ImageBackground`         | `ImageBackgroundInstance`         | `ImageBackground`                              |
| `KeyboardAvoidingView`    | `KeyboardAvoidingViewInstance`    | `KeyboardAvoidingView`                         |
| `Modal`                   | `ModalInstance`                   | `HostInstance`                                 |
| `Pressable`               | `PressableInstance`               | `HostInstance`                                 |
| `ProgressBarAndroid`      | `ProgressBarAndroidInstance`      | `HostInstance`                                 |
| `RefreshControl`          | `RefreshControlInstance`          | `RefreshControl`                               |
| `SafeAreaView`            | `SafeAreaViewInstance`            | `HostInstance`                                 |
| `ScrollView`              | `ScrollViewInstance`              | `HostInstance + ScrollViewImperativeMethods`   |
| `SectionList`             | `SectionListInstance`             | `SectionList<>`                                |
| `StatusBar`               | `StatusBarInstance`               | `StatusBar`                                    |
| `Switch`                  | `SwitchInstance`                  | `HostInstance`                                 |
| `Text`                    | `TextInstance`                    | `HostInstance`                                 |
| `TextInput`               | `TextInputInstance`               | `ReactNativeElement + imperative methods`      |
| `TouchableHighlight`      | `TouchableHighlightInstance`      | `HostInstance`                                 |
| `TouchableNativeFeedback` | `TouchableNativeFeedbackInstance` | `TouchableNativeFeedback`                      |
| `TouchableOpacity`        | `TouchableOpacityInstance`        | `HostInstance`                                 |
| `View`                    | `ViewInstance`                    | `HostInstance`                                 |
| `VirtualizedList`         | `VirtualizedListInstance`         | `React.ElementRef<VirtualizedListType>`        |
| `VirtualizedSectionList`  | `VirtualizedSectionListInstance`  | `React.ElementRef<VirtualizedSectionListType>` |

Components without ref support (`InputAccessoryView`, `TouchableWithoutFeedback`, `experimental_LayoutConformance`) are excluded.

## Drawbacks

- **Still two ways to get the instance type**: Even with `*Instance` exports, `React.ComponentRef<typeof View>` remains valid and produces the same type. We are adding a convenient alias, not removing the utility type approach. In Flow, `React.ElementRef<typeof View>` will remain the idiomatic pattern since Flow does not benefit from dedicated exports the same way (component syntax handles this natively).
- **Surface area**: This adds ~24 new type exports to the `react-native` package. However, these are simple type aliases with no runtime cost, and they follow a mechanical pattern that is easy to maintain as components are added or removed.
- **Naming divergence from web**: The web platform uses `HTML*Element` (e.g. `HTMLDivElement`). We use `*Instance` instead. This is intentional — React Native elements are not HTML elements, and `*Element` would be misleading. However, it means the naming convention is not transferable between React DOM and React Native.

## Alternatives

### Alternative A: Preserve class-like TypeScript declaration merging

The hand-written TypeScript types use class declarations, which make `View` work as both a value and an instance type. We could preserve this behavior in the generated types using TypeScript declaration merging:

```ts
// Value export
export { default as View } from "./Libraries/Components/View/View";
// Merged type export (instance)
export type View = ViewInstance;
```

This preserves the `useRef<View>(null)` shorthand that existing TypeScript users rely on.

**Why avoid this:**

- Cannot be represented in Flow. Flow does not support declaration merging, so this pattern would diverge the TypeScript types from the Flow source — undermining the goal of auto-generated types.
- Requires custom transform steps to emit the merged declaration without breaking the Flow source (see [abandoned approach in #56673](https://github.com/facebook/react-native/pull/56673)).
- The merged `View` type is a magic implicit — it is not obvious that `View` the type differs from `View` the value. Dedicated `ViewInstance` is explicit about what it represents.

### Alternative B: Require `React.ComponentRef` utility type only (existing state)

We could document `React.ComponentRef<typeof View>` as the canonical pattern and not add any new exports:

```ts
const viewRef = useRef<React.ComponentRef<typeof View>>(null);
```

**Why avoid this:**

- Verbose — especially for the most common ref types (`View`, `TextInput`).
- The syntax differs between Flow (`React.ElementRef`) and TypeScript (`React.ComponentRef`), making cross-language documentation harder.
- No parity with web, which provides dedicated element types (`HTMLDivElement` etc.) rather than requiring utility type wrappers.
- `HostInstance` still exists as a tempting but overly broad alternative — `*Instance` types channel developers toward the right type for each component.

## Adoption strategy

### Migration guidance

The migration from existing patterns is mechanical:

| Before                                                  | After                        |
| ------------------------------------------------------- | ---------------------------- |
| `useRef<View>(null)`                                    | `useRef<ViewInstance>(null)` |
| `useRef<React.ComponentRef<typeof View>>(null)`         | `useRef<ViewInstance>(null)` |
| `useRef<HostInstance>(null)` (for a specific component) | `useRef<ViewInstance>(null)` |
| `Ref<Animated.LegacyRef<View>>`                         | `Ref<ViewInstance>`          |

### Reducing the net breaking change of the strict types migration

**What to know**: This proposal is purely additive on its own — new type exports alongside existing ones. No existing code breaks today. However, the motivation is forward-looking: when developers adopt the [Strict TypeScript API](https://reactnative.dev/docs/strict-typescript-api), every `useRef<View>` in their codebase will become a type error because `View` is no longer a class. Without `*Instance` types, the migration path is `useRef<React.ComponentRef<typeof View>>` — verbose, unfamiliar, and a significant friction point across potentially hundreds of call sites.

By shipping `*Instance` types ahead of the Strict TypeScript API migration, the upgrade path becomes `useRef<View>` → `useRef<ViewInstance>` — a minimal, mechanical change that preserves the mental model of "one type per component for refs."

### Tooling

The `useRef<View>` → `useRef<ViewInstance>` transformation can be handled by the `migrate-to-strict-api` AI skill being developed for the broader [Strict TypeScript API migration](https://reactnative.dev/docs/strict-typescript-api).

## How we teach this

### Documentation

The [Direct Manipulation](https://reactnative.dev/docs/the-new-architecture/direct-manipulation-new-architecture) docs already use `useRef<TextInput>` in some examples. All ref-related documentation and examples should be updated to use `*Instance` types as the recommended pattern.

### Mental model

> Every built-in component has a corresponding `*Instance` type for its ref. Use `ViewInstance` for `<View>`, `TextInputInstance` for `<TextInput>`, etc. The same instance type works for `Animated` variants.

This is a simple rule that works across all components and both languages. Developers familiar with web's `HTMLDivElement` / `HTMLInputElement` pattern will recognize the structure immediately, even though the naming differs.

### Deprecation of `HostInstance` for direct use

We should consider soft-deprecating direct use of `HostInstance` in application code (via documentation, not a compiler warning). `HostInstance` is the correct base type for library authors writing component-agnostic utilities, but application code should prefer the specific `*Instance` type to preserve access to imperative methods.

## Unresolved questions

- **Should `HostInstance` be deprecated for direct use in application code?** It remains useful for library authors who need a generic "any native element" type, but using it for specific component refs loses type safety.
- **Future `*Element` naming?** If React Native moves closer to web standards (e.g. via `ReactNativeElement` as the base class), should we revisit `*Instance` → `*Element` naming for consistency? The `Instance` naming is well-established in the React Native ecosystem today, but this is worth considering for the long term.
- **Flow adoption**: Flow's component syntax handles ref types natively via `React.ElementRef`. Should we also export `*Instance` types in Flow for cross-language consistency, or let Flow continue to derive them?
