---
title: Library codegen as prefab on Android
author:
  - Wojciech Lewicki
date: 05.12.2025
---

# RFC0000: Library codegen as prefab on Android

## Summary

On Android, each library with Fabric components and/or Turbomodules produces codegen code during its build. This code, even though it resides in the `build` folder of the library, is not built by the library itself, but instead by the target in which the RN code resides (typically the `:app` target).

This proposal discusses an opt-in option for libraries to build their codegen and expose it via prefab to RN core, which consumes it. The purpose is to speed up the build process and move responsibilities to the correct place.

## Motivation

There are two main objectives of this endeavor:

- **Moving responsibility for building codegen bindings** to the entity that owns this code. Currently, even though codegen is the interface that each library exposes to RN core for their components and modules, the library itself is unaware that this code exists and bears no responsibility for building it. Meanwhile, when using custom shadow nodes, libraries must implement workarounds to make it work correctly (see: https://github.com/software-mansion/react-native-screens/blob/1746a584eed0ed1b1e9ac08909e2eec3789a3c0f/android/src/main/jni/CMakeLists.txt)

- **Speeding up the build process on Android.** Currently, all C++ codegen code is built and linked directly by RN core. Build time grows linearly with the number of libraries and is not parallelized. By moving the build process to individual libraries, the work can be distributed across them and performed simultaneously, with core only linking the artifacts at the end. This will also benefit projects like [RNRepo](https://rnrepo.org/), which aim to reduce build time by providing pre-built artifacts. Exposing library C++ code via prefab is the supported approach for achieving this.

## Detailed design

Since this solution changes how the build system works, I suggest introducing it as opt-in initially, allowing libraries to migrate gradually and in a controlled manner.

The project consists of two main parts—core and library code changes—and an interface to communicate this information.

- **First**, additional code is needed in RN core to expose a way for libraries to indicate they are providing their codegen as a prefab, so core does not need to build it. When opted in, core will look for the codegen prefab artifact instead. The library name and codegen section are already available in [the code that generates these bindings](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/tasks/GenerateAutolinkingNewArchitecturesFileTask.kt). Since configuration is now provided to each target during the build process (including [`ReactNative-application.cmake`](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/react-native/ReactAndroid/cmake-utils/ReactNative-application.cmake)), it should be generated as part of each library's [`CMakeLists.txt`](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/react-native-codegen/src/generators/modules/GenerateModuleJniH.js).

- **Second**, additional code is needed in each library to build its codegen and expose it via prefab. This can be approached manually or automatically.

  - **Manual option:** Each library augments its `build.gradle` to build codegen and expose prefab. This requires approximately 40 lines of code in the basic case but provides full control to the library. It also handles custom setups more easily.

  - **Automatic option:** Most likely involves creating a new plugin or extending the existing `com.facebook.react` plugin. This plugin would add the necessary setup transparently to the library, which is suitable for most cases since it's an implementation detail of RN-library integration.

  Both options are valuable: most libraries can simply apply the plugin without maintenance concerns. However, some libraries already expose C++ code or prefabs (like [react-native-reanimated](https://github.com/software-mansion/react-native-reanimated/blob/222f54bb9cca7cd8be2d3f29145525a39a6b3811/packages/react-native-reanimated/android/build.gradle#L177-L181)) or provide custom shadow nodes (like [react-native-screens](https://github.com/software-mansion/react-native-screens/blob/0f70e5dad6fa60ecc98155c9efe71c3c7833f989/react-native.config.js#L20)). The latter case may be harder to automate, further demonstrating that the current system relies on hacks and requires custom workarounds.

- **Interface options:**

  - Add a flag in the library's `react-native.config.js`, e.g., `hasCodegenPrefab`. RN core reads this, and when set to `true`, it switches from building codegen to looking for a prefab in the library.
  - Automatically look for a prefab in each library first, then fall back to building codegen if unavailable.
  - Pass the information through the plugin mentioned in the library section above.

## Drawbacks

As with any system-level change, potential bugs are a concern. Making this opt-in initially mitigates this risk by allowing thorough testing on critical ecosystem libraries and addressing issues early.

This may place additional burden on libraries, though one could argue this responsibility should have belonged to each library from the start.

## Adoption strategy

This change only affects library authors. Depending on the approach chosen, they will either require no action or can follow clear opt-in instructions. These instructions should be available on the website with examples from early-adopting libraries. The solution must remain backward compatible.

## How we teach this

Provide clear communication in the changelog, framing this as a system improvement that results in faster builds.

## Unresolved questions

- If we create a plugin for libraries, should we reuse the existing plugin or create a new one, which would require changes to each library?
