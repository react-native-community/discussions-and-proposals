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

- Version 0.77: Remove the [`jsc-android` npm dependency](https://github.com/facebook/react-native/blob/2d337efc23c662143b3f39a6c994d80fec047054/packages/react-native/package.json#L132) from React Native and instead download the artifact from Maven Central. When using Hermes, downloading the unused npm dependency is unnecessary and removing it saves about 32 MB of uncompressed size.
- Version 0.78 or 0.79:  Introduce a new `react-native-javascriptcore` npm package with a newer version of JavaScriptCore for Android. In this version, we will only provide the [Intl variant](https://github.com/react-native-community/jsc-android-buildscripts/blob/9c61fece4753902a2cd6d29dfa46b7b521f0c821/README.md#international-variant), as newer JavaScriptCore versions do not support disabling Intl. The existing JSCRuntime from the core should still function with this version but will display a deprecation warning during build time. A challenge will be managing the core JSCRuntime alongside the third-party JSCRuntime from `react-native-javascriptcore`; further study is required.
- Version 0.80: Completely remove JavaScriptCore from the core:
  - Eliminate `JSCRuntime` code.
  - Deprecate Gradle's `hermesEnabled` and Podfile's `hermes_enabled`, making them no-ops.
  - Update documentation at https://reactnative.dev/docs/hermes.

## Motivation

Since Hermes has become the dominant JavaScript engine in the React Native ecosystem, it is inefficient to include unused JavaScriptCore code and binaries in the core. By extracting JavaScriptCore to a separate, third-party maintained library, we can streamline the core and reduce unnecessary bloat.

## Adoption strategy

We will offer a drop-in replacement with the `react-native-javascriptcore` npm package while maintaining JSC support in the core for two additional React Native releases to ensure a smooth transition.

## How we teach this

We will announce the upcoming changes in the version 0.77 release notes and provide guidance for developers on migrating to the new `react-native-javascriptcore` package.
