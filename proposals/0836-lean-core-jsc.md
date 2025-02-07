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

- Version 0.77: Remove the [`jsc-android` npm dependency](https://github.com/facebook/react-native/blob/2d337efc23c662143b3f39a6c994d80fec047054/packages/react-native/package.json#L132) from React Native and instead download the artifact from Maven Central. When using Hermes, downloading the unused npm dependency is unnecessary and removing it saves about 32 MB of uncompressed build time size.
- Version 0.80: Introduce a new `react-native-javascriptcore` npm package with a newer version of JavaScriptCore for Android. In this version, we will only provide the [Intl variant](https://github.com/react-native-community/jsc-android-buildscripts/blob/9c61fece4753902a2cd6d29dfa46b7b521f0c821/README.md#international-variant), as newer JavaScriptCore versions do not support disabling Intl. The existing JSCRuntime from the core should still function with this version but will display a deprecation warning during build time. A challenge will be managing the core JSCRuntime alongside the third-party JSCRuntime from `react-native-javascriptcore`; details will be mentioned in the [Mixing `JSCRuntime` from core and third-party](#mixing-jscruntime-from-core-and-third-party) section.
  - Could we support `react-native-javascriptcore` only on New Architecture mode?
- Version 0.82: Completely remove JavaScriptCore from the core:
  - Eliminate `JSCRuntime` code and drop `useThirdPartyJSC` support.
  - Update documentation at https://reactnative.dev/docs/hermes.
  - At this point, React Native will stop JSCRuntime testing from core. All the integration testing effort should move into the `react-native-javascriptcore` repository.

## Motivation

Since Hermes has become the dominant JavaScript engine in the React Native ecosystem, it is inefficient to include unused JavaScriptCore code and binaries in the core. By extracting JavaScriptCore to a separate, third-party maintained library, we can streamline the core and reduce unnecessary bloat.

## Adoption strategy

We will offer a drop-in replacement with the `react-native-javascriptcore` npm package while maintaining JSC support in the core for two additional React Native releases to ensure a smooth transition.

### Mixing `JSCRuntime` from core and third-party

Originally, the `hermesEnabled` in Android's **gradle.properties** and `hermes_enabled` in iOS's **Podfile** take two responsibilities. One is to create the Hermes bytecode bundle. The other is to choose JSCRuntime over HermesRuntime. During the transition period, we will introduce a `useThirdPartyJSC` flag for **gradle.properties** and `ENV['useThirdPartyJSC']` for CocoaPods. When `useThirdPartyJSC` is enabled, React Native will not link the `JSCRuntime` from core.

### Setup guide for `react-native-javascriptcore`

A rough idea to set up `react-native-javascriptcore`:

1. `yarn add react-native-javascriptcore`
2. Set `hermesEnabled=false`
3. Add `useThirdPartyJSC=true` on React Native 0.80~0.82
4. On Android, update MainApplication to support custom JSRuntimeFactory for JSC
5. TODO: On iOS, there are some challenges since `createJSRuntimeFactory()` requires C++ integration and we now have AppDelegate.swift. We could try to abstract the `createJSRuntimeFactory()` to an Objective-C interface first, so that people can pass their own JSRuntimeFactory through Swift. I will study some approaches and update this part later.

## How we teach this

We will announce the upcoming changes in the version 0.80 release notes and provide guidance for developers on migrating to the new `react-native-javascriptcore` package. After version 0.82, the core will no longer support JSCRuntime and no more integration testing will be done in the core. All the integration testing effort should move into the `react-native-javascriptcore` repository.
