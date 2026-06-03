---
title: Rename NativeDeviceInfo.getConstants API
author:
- Artur Morys-Magiera
date: 19.12.2025
---

# RFC0000: Rename `NativeDeviceInfo.getConstants` API

## Summary

This RFC proposes:
- renaming the existing `NativeDeviceInfo.getConstants` method to `NativeDeviceInfo.getInfo`, to better reflect that the returned values may change during the app's lifecycle
- renaming the internal type `DeviceInfoConstants` to `DeviceInfoValues` to better represent the dynamic nature of the information it holds

## Basic example

```typescript
 const NativeModule: Spec = TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
 
 const NativeDeviceInfo = {
+  /**
+   * Please note that on iOS, if the application is using SceneDelegate, the dimensions
+   * may not be constant, in which case calling this method returns the dimensions valid
+   * at the time of invocation.
+   * @deprecated Use `NativeDeviceInfo.getInfo()` instead; this method will be removed in a future version of React Native.
+   * @returns {DeviceInfoConstants} the dimensions at the moment of invocation
+   */
   getConstants(): DeviceInfoConstants {
     return NativeModule.getConstants();
   },
+  getInfo(): DeviceInfoConstants {
+    return NativeModule.getConstants();
+  },
 };
 
 export default NativeDeviceInfo;
```

## Motivation

As nowadays mobile apps' windows may be resized, both on Android and iOS, the current naming of the API `NativeDeviceInfo.getConstants` is misleading, as the term "constants" implies a fixed set of characteristics that do not change during the app's lifecycle. This used to be true, however, now it is not the case. Therefore, this RFC proposes that the original method be renamed to `NativeDeviceInfo.getInfo`, which would better reflect its purpose of retrieving the current device information, which may change over time.

Moreover, the current internal code uses a type `DeviceInfoConstants` that is also no longer accurate. This type should be renamed to `DeviceInfoValues` to better represent the dynamic nature of the information it holds.

## Detailed design

1. Deprecate the existing `NativeDeviceInfo.getConstants` method by adding a deprecation notice in its JSDoc comment, advising developers to use the new method instead.
2. Introduce a new method `NativeDeviceInfo.getInfo` that performs the same function as `getConstants`, returning the current device information.
   - The native Android infra already supports this; the iOS native infra caches the constants but this behaviour will be changed after #967 is completed.
3. Rename the internal type `DeviceInfoConstants` to `DeviceInfoValues` to reflect the dynamic nature of the data.
4. Update the documentation and RNTester to reference the new method and type names.

## Drawbacks

In the end of the process, removal of the current API alias will be a breaking change.

## Alternatives

An alternative approach is to keep the existing method & type names, which could be misleading for developers, implying the values are constant.

## Adoption strategy

Adoption should happen in two phases:
1. Deprecation: introduce the new method `getInfo` while keeping the existing `getConstants` method. Update documentation and examples to encourage developers to use the new method.
2. Removal: after a suitable deprecation period (e.g., one major release cycle), remove the `getConstants` method from the API.

At stage 1., we could introduce a codemod to automate migration.

## How we teach this

The documentation must be updated to reference the new method - `getInfo`. For the time of the deprecation period, both methods should be documented, with clear indications of which one is preferred.

## Unresolved questions

N/A
