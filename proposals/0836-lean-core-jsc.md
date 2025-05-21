---
title: Extract JSC (JavaScriptCore) from React Native Core
author:
  - Kudo Chien <kudo@expo.dev>
date: 2024-11-28
---

# RFC0836: Extract JSC (JavaScriptCore) from React Native Core

## Summary

This RFC proposes a Lean Core effort to extract JSC (JavaScriptCore) from React Native's core.

## Proposed Changes and Timeline

- **Version 0.78**: Remove the [`jsc-android` npm dependency](https://github.com/facebook/react-native/blob/2d337efc23c662143b3f39a6c994d80fec047054/packages/react-native/package.json#L132) from React Native and instead download the artifact from Maven Central. When using Hermes, downloading the unused npm dependency is unnecessary and removing it saves about 32 MB of uncompressed build time size.
- **Version 0.79**: Introduce the community [`@react-native-community/javascriptcore`](https://github.com/react-native-community/javascriptcore) package, allowing developers to begin migrating from core JSC to the community-maintained version. Using the core JSC will trigger a build-time warning about its upcoming removal:
  - React Native core will reduce its [testing support](https://github.com/reactwg/react-native-releases/blob/main/docs/guide-release-testing.md#dimensions-to-test) for JSC.
  - The community JSC package will include basic end-to-end tests.
- **Version 0.80**: The `@react-native-community/javascriptcore` package will offer a newer JSC version on Android that may support modern JavaScript language features, such as [`BigInt`](https://github.com/react-native-community/jsc-android-buildscripts/pull/169). In this updated `jsc-android`, only the [`Intl` variant](https://github.com/react-native-community/jsc-android-buildscripts/blob/9c61fece4753902a2cd6d29dfa46b7b521f0c821/README.md#international-variant) will be provided because newer JavaScriptCore versions do not support disabling `Intl` The core JSC remains available in this version with a build-time warning:
  - Update documentation at https://reactnative.dev/docs/hermes to recommend the community JSC.
- **Version 0.81 or 0.82**: Completely remove JSC from the core:
  - Eliminate `JSCRuntime` code and drop support for `useThirdPartyJSC`.

## Motivation

Since Hermes has become the dominant JavaScript engine in the React Native ecosystem, it is inefficient to include unused JavaScriptCore code and binaries in the core. By extracting JavaScriptCore to a separate, third-party maintained library, we can streamline the core and reduce unnecessary bloat.

## Adoption strategy

We will offer a drop-in replacement with the [`@react-native-community/javascriptcore`](https://github.com/react-native-community/javascriptcore) package while maintaining JSC support in the core for two additional React Native releases to ensure a smooth transition.

> [!NOTE]
> The community JSC will only support New Architecture mode.

### Mixing `JSCRuntime` from core and third-party

Originally, the `hermesEnabled` flag in Android's **gradle.properties** and `hermes_enabled` flag in iOS's **Podfile** served two purposes: creating the Hermes bytecode bundle and selecting JSCRuntime over HermesRuntime. During the transition period, we will introduce a `useThirdPartyJSC` flag in **gradle.properties** and `ENV['USE_THIRD_PARTY_JSC']` in CocoaPods. When `useThirdPartyJSC` is enabled, React Native will not link the core `JSCRuntime`.

### Setup guide for `@react-native-community/javascriptcore`

Refer to the [README from `@react-native-community/javascriptcore for setup instructions](https://github.com/react-native-community/javascriptcore/blob/main/README.md).

## How we teach this

- We will announce the upcoming changes in the version 0.79 release notes and provide guidance for developers on migrating to the new `@react-native-community/javascriptcore` package.
- A build-time warning in React Native will notify users about the upcoming lean core changes regarding JSC.
