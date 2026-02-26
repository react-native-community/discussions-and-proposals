---
title: New entry point for `devsupport` APIs
author:
- Alex Hunt
date: 2026-02-07
---

# RFC0985: New entry point for `devsupport` APIs

## Summary

Formally expose existing development-time JavaScript APIs for use by Frameworks and ecosystem tooling via a `react-native/devsupport` subpath export.

## Basic example

```js
import {
  HMRClient,
  openURLInBrowser,
  parseErrorStack,
  symbolicateStackTrace,
  XHRInterceptor, // @deprecated
} from 'react-native/devsupport';
```

## Motivation

**Why it matters:** Major ecosystem libraries currently deep-import unstable internal APIs for development features.

**The problem:** React Native's development utilities—HMRClient, parseErrorStack, PressabilityDebugView—are internal. This forces ecosystem authors to import from `react-native/Libraries/...` paths that can break between releases.

**Who's affected:**
- [Re.pack](https://github.com/callstack/repack) — Custom bundler requiring HMRClient for Hot Module Replacement
- [React Native Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler) — Needs PressabilityDebugView for custom Pressable debugging
- Framework authors building dev tooling and error reporting

**What this enables:** Stable, documented access to development utilities without relying on implementation details.

## Detailed design

Add a subpath export to `react-native/package.json`:

```json
  "exports": {
    "./devsupport": {
      "react-native-strict-api": "./src/devsupport.d.ts",
      "default": "./src/devsupport.js"
    }
  }
```

This is implemented as `react-native/unstable_devsupport` in [huntie/react-native@a3880cb](https://github.com/huntie/react-native/commit/a3880cb484886396bc76732e1a33d536e02de60d).

**Pros:**
- Simpler migration path for consumers
- No new package to publish or version
- Immediately available to bundler authors without an additional dependency

**Cons:**
- Keeps dev-only code in the main package
- The `unstable_` prefix used during initial rollout may deter adoption before graduation


### Proposed exports

| Export | Current location | Dependencies | Community request |
|--------|------------------|--------------|-------------------|
| `HMRClient` | `Libraries/Utilities/HMRClient.js` | LogBox, NativeRedBox, DevSettings, Platform, getDevServer, metro-runtime | [Re.pack](https://github.com/callstack/repack) |
| `openURLInBrowser` | `Libraries/Core/Devtools/openURLInBrowser.js` | getDevServer | — |
| `parseErrorStack` | `Libraries/Core/Devtools/parseErrorStack.js` | parseHermesStack, stacktrace-parser | [[1]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `symbolicateStackTrace` | `Libraries/Core/Devtools/symbolicateStackTrace.js` | getDevServer, fetch | — |
| `PressabilityDebugView` | `Libraries/Pressability/PressabilityDebug.js` | Pressability internals | [React Native Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler) [[2]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `XHRInterceptor` *(deprecated)* | `Libraries/Network/...` | TBD | — |

### Deferred exports (out of scope)

The following community-requested exports relate to renderer internals and view system APIs rather than development tooling. These may be better served by a separate `react-native/internals` subpath export for framework authors:

| Export | Current location | Requested by | Ref |
|--------|------------------|--------------|-----|
| `NativeText` | `Libraries/Text/TextNativeComponent.js` | Component libraries | [[3]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `customDirectEventTypes` | `Libraries/Renderer/shims/ReactNativeViewConfigRegistry.js` | [React Native Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler) | [[4]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `ReactNativeStyleAttributes` | `Libraries/Components/View/ReactNativeStyleAttributes.js` | [React Native Screens](https://github.com/software-mansion/react-native-screens) | [[5]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `AppContainer` | `Libraries/ReactNative/AppContainer.js` | [React Native Screens](https://github.com/software-mansion/react-native-screens) | [[6]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `findHostInstance_DEPRECATED` | `Libraries/Renderer/shims/ReactNative.js` | [React Native Reanimated](https://github.com/software-mansion/react-native-reanimated), [Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler), [Screens](https://github.com/software-mansion/react-native-screens) | [[7]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |
| `getInternalInstanceHandleFromPublicInstance` | `Libraries/ReactNative/ReactFabricPublicInstance/ReactFabricPublicInstance.js` | [React Native Reanimated](https://github.com/software-mansion/react-native-reanimated), [Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler), [Screens](https://github.com/software-mansion/react-native-screens) | [[8]](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) |

`getDevServer`? (Rozenite)

### Alternative design: `@react-native/devsupport` package

Instead of a subpath export, create a standalone `@react-native/devsupport` package.

**Pros:**
- Clear separation of dev-only code
- Potential bundle size win via tree-shaking in production
- Aligns with the existing `@react-native/*` package structure

**Cons:**
- Additional package to maintain and version
- Breaking change if later consolidated back
- `HMRClient` depends on react-native internals (`LogBox`, `NativeRedBox`, `DevSettings`, `Platform`), so the package would still require `peerDependencies: { "react-native": "*" }`, undermining the separation benefit

#### Extractability analysis

The `Libraries/Core/Devtools` folder (`parseErrorStack`, `symbolicateStackTrace`, `openURLInBrowser`, `parseHermesStack`, `getDevServer`) has minimal external dependencies and **could be fully extracted** to a separate package. However, `HMRClient`'s deep coupling to react-native internals means a standalone package cannot avoid a peer dependency on `react-native`.

## Drawbacks

The strict public API work must be completed before legacy deep imports can be removed.

## Unresolved questions

- Graduation criteria for dropping the `unstable_` prefix.
- Whether `getDevServer` and `parseHermesStack` should also be public exports.
- Should renderer/view system internals be exposed via `devsupport` or a separate `internals` subpath?

#### References

- [1–8] [Community Requested Root Exports spreadsheet](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0)
- https://github.com/facebook/react-native/pull/53081
