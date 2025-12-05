---
title: Library codegen as prefab on Android
author:
  - Wojciech Lewicki
date: 05.12.2025
---

# RFC0000: Library codegen as prefab on Android

## Summary

On Android, each library having Fabric components and/or Turbomodules produces codegen code during its build. This code, even though it lies in `build` folder of the library, is not built by the library, but instead by the target in which the RN code lies (typically `:app` target).
This proposal discusses an opt-in option for the libraries to be able to build their codegen and expose it via prefab to RN core which consumes it. The purpose of it is to speed up the build process and move the responsibilities to correct place.

## Motivation

There are two main objectives of this endeavour:

- Moving the responsibility of building codegen bindings to the subject that this code concerns - right now, even though codegen is the interface that each library exposes to RN core for their components and modules, the library itself is not even aware that this code exists and is not responsible for building it. At the same time, when using e.g. custom shadow nodes, it has to do the hacky shenanigans to make it work correctly (see: https://github.com/software-mansion/react-native-screens/blob/1746a584eed0ed1b1e9ac08909e2eec3789a3c0f/android/src/main/jni/CMakeLists.txt)
- Speeding up the build process on Android. Right now, all of the cpp codegen code is being built and linked directly by RN core. The amount of time necessary to do it grows linearly with the number of libraries and is not done in parallel. When moving the build process to the libraries, it can be spread across them and done simultaneously, and core is only meant to link the artifacts at the end. This will also benefit projects like [RNRepo](https://rnrepo.org/), which try to cut the build time by providing already built artifacts, and providing library's cpp code via prefab is the supported way of doing it.

## Detailed design

Since this solution changes how the build system work, I suggest integrating it as an opt-in at first, so we can move the libraries gradually and under control.

The project consists of two main parts, core and library code changes, and an interface to pass the information around.

- First is additional code needed in RN core, exposing a way for libraries to inform it that they are providing their codegen as a prefab, so that the core does not need to build it by itself. When opted-in, core code will look for the codegen prefab artifact instead. The name of library and its codegen part is available in [the code generating those bindings](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/tasks/GenerateAutolinkingNewArchitecturesFileTask.kt) already. Since we now provide all the configuration to each target as part of building process (whole [`ReactNative-application.cmake`](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/react-native/ReactAndroid/cmake-utils/ReactNative-application.cmake)), we should move it to being generated as part of each [`CMakeLists.txt`](https://github.com/facebook/react-native/blob/95cc1e76756ea0d67bfe20be35aff99c4028f2f2/packages/react-native-codegen/src/generators/modules/GenerateModuleJniH.js).

- Second is the additional code needed in each library that will make it build its codegen. This can be approached manually or automatically.

  - As for the manual option, each library would have to augment its `build.gradle` to build codegen and expose prefab. It is around 40 lines of code in the basic case but gives the full control of it to the library. It is also easier to handle custom setups such way.

  - Automatic option would most probably involve creating new plugin or adding more code to `com.facebook.react` one. Such plugin would add the necessary setup to the library, making it fully transparent to the library, which seems a good option for most cases, as it is kind of implementation detail of the RN-library integration.

Having both options seems like a good way to go since most libraries would just apply the plugin and not care about maintaining it. There are still some libraries that either expose cpp code already (and maybe a prefab with them too, like [react-native-reanimated](https://github.com/software-mansion/react-native-reanimated/blob/222f54bb9cca7cd8be2d3f29145525a39a6b3811/packages/react-native-reanimated/android/build.gradle#L177-L181)), or provide custom shadow nodes (like [react-native-screens](https://github.com/software-mansion/react-native-screens/blob/0f70e5dad6fa60ecc98155c9efe71c3c7833f989/react-native.config.js#L20)). For the second case it might be even harder to automate it, showing even more that the current system is far from flawless and any customizations work based on dirty hacks.

- As for the interface, there are probably a couple ways to do it:

  - Adding a flag in library's `react-native.config.js`, e.g. `hasCodegenPrefab`. RN core would read it, and when set to `true`, it will switch from building codegen to looking for a prefab in library.
  - Automatically starting with looking for a prefab in each library, and if it is not available, building codegen by itself.
  - Passing the information as part of plugin mentioned in the library part above.

## Drawbacks

As with every change of that type, the main drawback is the potential bugs coming with it. Making it as an opt-in at the beginning should mitigate that risk though by testing it thoroughly on crucial libraries in the ecosystem and addressing any problems early.

It potentially adds more burden to the libraries, but it might be argued that handling this should have always been the responsibility of each library.

## Adoption strategy

This change will only be affecting library authors. Depending on the path we take, they will either not need to do anything or follow clear instructions on how to opt-in to the new system. Those instructions should be available on the website with examples from libraries that already adopted the improvement. The solution should of course be backward compatible.

## How we teach this

Clear communication in the changelog. Communicating the solution as an improvement of the system which should resolve in faster builds.

## Unresolved questions

- If we go for a plugin for libraries, should we reuse the already available one or add another, which would imply necessity of changes to each library?
