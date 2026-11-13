---
title: Node-API addons in React Native
author:
- Kræn Hansen
date: 2026-11-09
---

# RFC0000: Node-API addons in React Native

## Summary

Let React Native apps load [Node-API](https://nodejs.org/api/n-api.html) addons, using the Node-API implementation Hermes already has. React Native would add one JavaScript function, `dlopen` exported from `react-native/node-api`. It would also implement what Hermes' Node-API implementation asks of its embedder: running async work on worker threads, calling into JavaScript from other threads, and reporting fatal exceptions. Hermes would ship an engine-agnostic `node-api` library for addons to link against, so the only library an addon's binary references is `node-api`, never the engine.

Everything that decides how addons are named, packaged, found and cached stays outside core. That keeps core lean, and leaves those parts free to evolve in community projects.

A working implementation for Android and iOS is in [react/react-native#59894](https://github.com/react/react-native/pull/59894).

## Basic example

An addon, written against the Node-API headers:

```c
#include <node_api.h>

static napi_value Hello(napi_env env, napi_callback_info info) {
  napi_value result;
  napi_create_string_utf8(env, "Hello from Node-API", NAPI_AUTO_LENGTH, &result);
  return result;
}

NAPI_MODULE_INIT() {
  napi_property_descriptor hello = {"hello", NULL, Hello, NULL, NULL, NULL, napi_default, NULL};
  napi_define_properties(env, exports, 1, &hello);
  return exports;
}
```

Built with CMake, the same way in an app as in a prebuild:

```cmake
find_package(node-api REQUIRED CONFIG)
add_library(my_addon SHARED addon.c)
target_link_libraries(my_addon PRIVATE node-api::node-api)
```

Loaded from JavaScript:

```js
import {Platform} from 'react-native';
import {dlopen} from 'react-native/node-api';

const addon = {exports: {}};
dlopen(
  addon,
  Platform.select({
    android: 'libmy_addon.so',
    ios: '@rpath/my_addon.framework/my_addon',
  }),
);
addon.exports.hello(); // "Hello from Node-API"
```

## Motivation

Node-API started as Node.js' API for native addons, and has become a standard way for native code to talk to JavaScript engines. Node.js, Deno and Bun implement it, and many npm packages ship Node-API addons. Compared with JSI, React Native's current way to call native code:

- **It's a stable C ABI.** An addon built once works with any engine and version that implements the Node-API version it targets. JSI is a C++ API without a stable ABI, so JSI code is compiled against the exact React Native version it runs with.
- **It allows prebuilt binaries.** Besides surviving engine updates without a rebuild, a stable ABI lets libraries ship compiled addons instead of making every app build them from source.
- **It isn't specific to React Native.** The same addon source code builds for Node.js, so it can be developed and tested without a simulator or device, and the existing tooling and bindings (for example [node-addon-api](https://github.com/nodejs/node-addon-api) for C++ and [napi-rs](https://napi.rs) for Rust) work as they are. Many addons already exist in the ecosystem, for example implementations of web standards, which React Native apps could use too.

Hermes now implements Node-API itself ([`API/napi`](https://github.com/meta/hermes/tree/51ae2a08ce8cb06fcfa718cddab9c94ea9eee7f6/API/napi) on `static_h`, covering Node-API v10). What React Native apps are missing is the rest:

1. A way to load an addon into the app's runtime.
2. The host integration Node-API needs from its embedder: running async work on worker threads, calling into JavaScript from other threads, and reporting fatal errors.
3. An engine-agnostic library for addons to link against, with a name that doesn't change.

[react-native-node-api](https://github.com/callstackincubator/react-native-node-api) provides all three outside core today, and shows what it costs: it vendors its own build of Hermes, has apps build React Native from source on Android, and links addons against a shim (`weak-node-api`) that forwards every Node-API call through a function table injected at startup. Only part 1 and part 2 need React Native, and they're small. Part 3 belongs with the engine.

Constraints for a design:

- **Addons shouldn't depend on the engine's library name.** Addons are built independently of apps. React Native 0.82 renamed `libhermes.so` to `libhermesvm.so`, and a rename like that would break every published addon that linked it. For app developers, the engine has also always been an implementation detail behind JSI.
- **Core shouldn't decide how addons are packaged or found.** Naming schemes, `require()` integration and caching are still evolving in the community, and don't have to become core API. Keeping them outside keeps core lean and leaves room to experiment, while core stays genuinely useful for addons whose build is driven by the app's build tooling.
- **It should be opt-in, with a small, measured cost to apps that don't use it.** React Native leaves it out unless an app enables it (see Enabling Node-API), but with prebuilt Hermes, Node-API is still compiled into the `hermesvm` every app ships. See Binary size under Drawbacks.

## Detailed design

### JavaScript API

```ts
// react-native/node-api

/**
 * Loads a Node-API addon into the calling JavaScript runtime, like Node's
 * `process.dlopen()`.
 *
 * Opens the shared library at `filename`, calls the addon's module
 * initializer with a new `napi_env` and an empty exports object, and sets
 * `module.exports` to the value it returns. Nothing is cached: loading the
 * same addon again initializes it again, in a new env. Each env is torn down
 * with the runtime, which runs the addon's env cleanup hooks.
 *
 * @param module An object whose `exports` property is set to the addon's
 *   exports.
 * @param filename Passed to the system's `dlopen()` as is, for example
 *   `libmy_addon.so` on Android or `@rpath/my_addon.framework/my_addon` on
 *   Apple platforms.
 * @throws If the JavaScript engine doesn't implement Node-API, or if the
 *   addon can't be loaded or throws while initializing.
 */
export function dlopen(module: {exports: unknown}, filename: string): void;
```

`dlopen` behaves like Node's [`process.dlopen(module, filename)`](https://nodejs.org/api/process.html#processdlopenmodule-filename-flags), with nothing added, so a difference from Node is a bug:

- It sets `module.exports` to the addon's exports.[^exports]
- `filename` is passed to the system's `dlopen` as is: `lib<name>.so` on Android, and `@rpath/<name>.framework/<name>` on Apple platforms.
- Nothing is cached. Loading an addon twice initializes it twice, as in Node. Caching belongs in a higher-level function, the way `require` sits above `process.dlopen` in Node.
- `flags` is left out for now, and can be added later without breaking anyone.
- It throws if the JavaScript engine doesn't provide Node-API.

It's a subpath export because `dlopen` is too ambiguous as a top-level `react-native` export. It's unflagged: apps and libraries opt in by calling it, and its surface follows an existing standard.

### Native module

`dlopen` is backed by a C++ TurboModule, `NativeNodeApiCxx`, in `ReactCommon/react/nativemodule/nodeapi`:

- **One `napi_env` per `dlopen` call**, as in Node. Each env is owned by the runtime and torn down with it, which runs the addon's env cleanup hooks. After a reload, addons are loaded into the new runtime with new envs.
- **`NodeApiEngine`** is the part that depends on the engine's embedder API: creating an env and loading an addon. Hermes implements it with `hermes_napi_create_env` and `hermes_napi_load_module`. Supporting another engine means implementing this interface, not changing the module.
- **`NodeApiHost`** is what the engine needs from React Native:
  - a worker pool for `napi_queue_async_work`, sized like libuv's;
  - dispatch onto the JS thread for thread-safe functions, through the runtime's `CallInvoker`;
  - routing `napi_fatal_exception` to `ErrorUtils.reportFatalError`, React Native's equivalent of Node's `'uncaughtException'`.

`HermesInstance` registers the module when it creates a runtime, with `registerCxxModuleToGlobalModuleMap`. It's only built into Hermes' part of React Native (`libhermestooling.so` on Android), so apps using another engine don't carry it, and `react-native/node-api` throws there.

Fatal Node-API errors (`napi_fatal_error`) go through Hermes' fatal handler, which reports them through React Native's logger since [#59729](https://github.com/react/react-native/pull/59729) and [#59861](https://github.com/react/react-native/pull/59861).

### Linking: the `node-api` library

Addons link against a library named `node-api`, which forwards to the engine and contains no code at runtime. The only library an addon's binary then references is `node-api` (its `DT_NEEDED` entry on Android, its `LC_LOAD_DYLIB` load command on Apple platforms), whichever engine provides it.

- **Android:** two builds of `libnode-api.so` with the same soname:
  - a) A stub for linking only, in the AAR's prefab module. It defines every Node-API function, with bodies that abort.
  - b) An empty library with a `DT_NEEDED` on `libhermesvm.so`, packaged into the app.

  One library can't do both. When linking, the NDK passes `--no-undefined`, and lld doesn't resolve symbols through a dependency's `DT_NEEDED`, so a) has to define them. At runtime, Android's dynamic linker would then choose the stub's definitions over `libhermesvm.so`'s, so b) can't define them. ELF has no re-export like Apple's (below): its closest equivalent, a filter library (`DT_FILTER`), is ignored by Android's dynamic linker[^bionic]. The NDK's own sysroot libraries work the same way: they're stubs generated from symbol maps, used only for linking[^ndkstubgen].
- **Apple platforms:** `node-api.xcframework`, with the same slices as `hermesvm.xcframework`. Each slice is linked with ld64's `-reexport_framework hermesvm`, which records an `LC_REEXPORT_DYLIB` load command: everything `hermesvm` exports counts as exported by `node-api`. Both the static linker and dyld follow it, so one build works for linking and at runtime. Apple's umbrella frameworks are built the same way.
- **Headers:** Hermes vendors the Node.js headers in `include/hermes/napi`. The library exposes that directory, so addons `#include <node_api.h>`.

The proposal is for Hermes to ship it, because `node-api` has to match the `hermesvm` it ships with: a) has to declare exactly what it exports, and b) names its soname. On Apple platforms, the one framework does both, re-exporting `hermesvm` by its install name. When a new Node-API version adds functions, Hermes can then update `node-api` together with its implementation, independently of React Native. This has been proposed to the Hermes team in [meta/hermes#1074](https://github.com/meta/hermes/discussions/1074#discussioncomment-19478312).

React Native makes it easy to use from an app:

- **Android:** the React Native Gradle plugin writes a `node-api-config.cmake` that defines `node-api::node-api`, and passes `-Dnode-api_DIR=<dir>` to the app's native build and every library's, the way it already [adds `-DRN_STRICT_API`](https://github.com/react/react-native/blob/f480836ad8e56ac53499c81f7cf0a186159711e5/packages/gradle-plugin/react-native-gradle-plugin/src/main/kotlin/com/facebook/react/utils/NdkConfiguratorUtils.kt#L60-L61). This way, libraries with their own `externalNativeBuild` don't need to depend on Hermes.
- **CocoaPods and SwiftPM:** a `node-api` pod and product.

> [!NOTE]
> Prebuilt addons are out of scope for this RFC. They don't need React Native: build tools can link against their own `node-api` stub with the same names (for example `node-api-stub` in react-native-node-api), and use the same `find_package(node-api)` and `node-api::node-api`.

### Enabling Node-API

Node-API is off by default[^naming]. Apps that load addons enable it, and build React Native from source:

- **Android:** the Gradle property `nodeApiEnabled=true` (or `react.nodeApiEnabled=true`).
- **CocoaPods and SwiftPM:** the environment variable `RCT_NODE_API_ENABLED=1` when running `pod install` or resolving the package.

This defines `RCT_NODE_API_ENABLED`, which compiles in the module and its registration in `HermesInstance`, and passes the switch to the app's own native build, so the app can build its addons. A Hermes built from source gets `HERMES_ENABLE_NAPI` to match, and React Native builds against a Hermes compiled without Node-API when the switch is off. Without it, `dlopen` throws, as it does with another engine. The prebuilt React Native artifacts (the AAR and `React.xcframework`) don't include Node-API, and `pod install` warns when the switch is set with a prebuilt `React.xcframework`.

### What stays outside core

- How addon files are named and laid out, and building the `filename` passed to `dlopen`.
- `require()` integration and caching[^babel].
- Autolinking addons into apps, and the prebuild tooling.

These can keep evolving in community projects, on top of `dlopen`.

### What this asks of Hermes

1. Ship Node-API in the Hermes builds React Native uses. `API/napi` is on `static_h`, but not yet in a release.
2. Ship the `node-api` library and the headers in `include/hermes/napi` with the Android and Apple artifacts, and keep the name stable. This is proposed in [meta/hermes#1074](https://github.com/meta/hermes/discussions/1074#discussioncomment-19478312).

## Drawbacks

- **New public API.** It's a single function, but it's a commitment, and so are the `node-api` library names once addons ship against them.
- **Hermes only, for now.** Apps using another engine get an error from `dlopen`. Other engines could support it by implementing `NodeApiEngine` and shipping a `node-api` library.
- **Threading code in core.** The host's worker pool and cross-thread dispatch have to be maintained in React Native, and have to stay correct across reloads and teardown.
- **Binary size.** Node-API is built into `hermesvm` by default (`HERMES_ENABLE_NAPI`), so apps using prebuilt Hermes carry it whether or not they load addons. The module is only compiled into React Native for apps that enable it (see Enabling Node-API). Measured on release arm64 builds of RNTester, with and without both (and without the example addon), using React Native's release settings:

  | RNTester, release arm64 | Grows by | Compressed (deflate -9) |
  | --- | --- | --- |
  | Android APK | 131,072 bytes (+0.62%) | 54,784 bytes (+0.49%) |
  | iOS app | 113,304 bytes (+0.80%) | 44,122 bytes (+0.61%) |

  RNTester is a large app, so the percentages would be higher for a minimal one. Against React Native's own native libraries on Android (11.6 MB stripped: `libreactnative`, `libhermesvm`, `libhermestooling`, `libjsi`, `libfbjni` and `libc++_shared`), the increase is 1.2%. By library, stripped:

  | Library | Grows by | gzipped |
  | --- | --- | --- |
  | `hermesvm`, Android | 93,960 bytes (+3.3%) | 36,647 bytes |
  | `hermesvm`, iOS | 92,784 bytes (+3.5%) | 33,135 bytes |
  | The module, in `libhermestooling.so` (Android) | 48,016 bytes | 18,133 bytes |
  | The module, in the app binary (iOS) | 20,520 bytes | 10,983 bytes |

  <details>
  <summary>How this was measured</summary>

  Each number compares two builds that differ only in the change being measured, from the implementation at [`95a798ebaf`](https://github.com/react/react-native/pull/59894/commits/95a798ebafb402d5c17b2aa1b7ae97e79d305bec) and Hermes at `51ae2a08ce8cb06fcfa718cddab9c94ea9eee7f6` (`static_h`), built from source.

  | Release arm64 | Without Node-API | With Node-API | Difference |
  | --- | --- | --- | --- |
  | RNTester APK | 21,108,900 | 21,239,972 | +131,072 (+0.62%) |
  | The same, deflate -9 | 11,189,692 | 11,244,476 | +54,784 (+0.49%) |
  | RNTester app (iOS) | 14,083,480 | 14,196,784 | +113,304 (+0.80%) |
  | The same, deflate -9 | 7,286,105 | 7,330,227 | +44,122 (+0.61%) |
  | `libhermesvm.so` | 2,818,632 | 2,912,592 | +93,960 (+3.3%) |
  | `hermesvm.framework` | 2,656,320 | 2,749,104 | +92,784 (+3.5%) |
  | `libhermestooling.so` | 150,488 | 198,504 | +48,016 |
  | RNTester app binary (iOS) | 6,152,480 | 6,173,000 | +20,520 |
  | React Native's native libraries (Android) | 11,569,264 | 11,711,240 | +141,976 (+1.23%) |
  | The same, gzip -9 | 4,135,953 | 4,190,733 | +54,780 (+1.32%) |

  **`hermesvm` on Android.** ReactAndroid's release variant of Hermes (`MinSizeRel`, `-flto=thin`, `-Os`, `HERMES_ENABLE_DEBUGGER=False`, Intl on, `HEAP_HV_PREFER32`):

  ```sh
  ./gradlew ':packages:react-native:ReactAndroid:hermes-engine:buildCMakeMinSizeRel[arm64-v8a][hermesvm]' \
    -PreactNativeArchitectures=arm64-v8a
  ```

  Then again with `"-DHERMES_ENABLE_NAPI=OFF"` added to the `release` arguments in `ReactAndroid/hermes-engine/build.gradle.kts`. Note that `buildCMakeRelease` builds the *debug* variant, with the debugger on.

  **`hermesvm` on iOS.** The flags of Hermes' `utils/build-apple-framework.sh` with `BUILD_TYPE=Release` (`MinSizeRel`, debugger off, Intl on, no LTO), for `iphoneos` arm64 with a deployment target of 15.1, built with Xcode 26.2:

  ```sh
  cmake -S sdks/hermes -B build_iphoneos -G Ninja \
    -DHERMES_APPLE_TARGET_PLATFORM=iphoneos -DCMAKE_OSX_ARCHITECTURES=arm64 \
    -DCMAKE_OSX_DEPLOYMENT_TARGET=15.1 -DCMAKE_BUILD_TYPE=MinSizeRel \
    -DHERMES_ENABLE_DEBUGGER=false -DHERMES_ENABLE_INTL=true \
    -DHERMES_BUILD_APPLE_FRAMEWORK=true -DHERMES_BUILD_SHARED_JSI=false \
    -DHERMES_ENABLE_BITCODE=false -DHERMES_ENABLE_TEST_SUITE=false \
    -DHERMES_EXTRA_LINKER_FLAGS="LINKER:-ld_classic" \
    -DCMAKE_C_FLAGS=-gdwarf -DCMAKE_CXX_FLAGS=-gdwarf \
    -DJSI_DIR=sdks/hermes/API/jsi -DIMPORT_HOST_COMPILERS=<host hermesc>/ImportHostCompilers.cmake \
    -DHERMES_ENABLE_NAPI=ON  # and OFF
  cmake --build build_iphoneos --target hermesvm
  ```

  **The module.** ReactAndroid's release native build (`RelWithDebInfo`), with and without the module: for the build without it, `$<TARGET_OBJECTS:react_nativemodule_nodeapi>` was removed from `hermestooling`, along with the registration in `HermesInstance.cpp` and `HermesNodeApiEngine.cpp`.

  ```sh
  ./gradlew ':packages:react-native:ReactAndroid:buildCMakeRelWithDebInfo[arm64-v8a]' \
    -PreactNativeArchitectures=arm64-v8a
  ```

  **The module on iOS.** It's linked statically into the app (CocoaPods' default static libraries, Release's `-dead_strip`), so RNTester was built in Release for `iphoneos` arm64, once as is and once without the registration in `HermesInstance.cpp`. Nothing references the module then, so the linker drops it: `nm` finds no `NodeApi` symbols in that binary.

  ```sh
  xcodebuild -workspace RNTesterPods.xcworkspace -scheme RNTester -configuration Release \
    -sdk iphoneos -destination 'generic/platform=iOS' ARCHS=arm64 CODE_SIGNING_ALLOWED=NO build
  ```

  **The apps.** On Android, RNTester's release APK for arm64, built with `./gradlew :packages:rn-tester:android:app:assembleRelease -PreactNativeArchitectures=arm64-v8a`, once as is and once with both of the changes above (Hermes without Node-API, and without the module). Both leave out the example addon, which isn't part of React Native's cost and can't link without Node-API. AGP stores native libraries uncompressed and aligned in the APK, which is why its growth is a round 131,072 bytes. On iOS, the Release `RNTester.app` from the builds above, with the matching stripped `hermesvm` swapped into `hermesvm.framework` in each, and `nodeapiexample.framework` removed. The compressed sizes recompress every entry of the APK, or of the zipped app, with deflate at level 9, as an estimate of download size.

  **Sizes.** Android libraries were stripped with the NDK's (28.2) `llvm-strip --strip-unneeded`, the iOS binaries with `strip -x`, and download size estimated with `gzip -9`. `llvm-nm -D` and `nm -gU` confirmed the builds with Node-API export all of its functions, so LTO doesn't remove them, and the builds without export none. The total for React Native's native libraries adds `libreactnative.so`, `libjsi.so`, `libfbjni.so` and `libc++_shared.so` from the same release build to the two pairs above.

  </details>
- **Two native extension models.** Library authors would choose between JSI+TurboModules and the alternative of Node-API. They serve different needs, but the documentation has to explain when to use which.
- **Not yet verified on the App Store.** Frameworks that re-export others are common, Apple's own umbrella frameworks among them, but no app shipping `node-api.xcframework` has been through App Store validation yet.

## Alternatives

- **Keep it all in user space.** react-native-node-api shows it works, but it needs a vendored Hermes and a shim library. The library name it gives addons is stable within that project, but not for React Native core.
- **React Native ships `node-api` (the library addons link against) instead of Hermes.** React Native would own the stable name whatever the engine is, but would have to re-package Hermes' headers and export list on every upgrade. Addons wouldn't notice the difference, because the library's name stays `node-api`. This is the fallback if Hermes prefers not to ship it.
- **Move Hermes' Node-API implementation into its own `node-api` library.** That's cleaner, but `hermesNapi` is built on VM internals that `hermesvm` doesn't export. Exporting them would be a large change to Hermes' API surface, and is unlikely to land.
- **Link addons against `hermesvm` directly.** It works today, but ties every addon to Hermes and to its library name, and exposes addons to symbols that aren't ABI-stable and that they don't need.
- **Wait for Node-API vtables** ([nodejs/node#60916](https://github.com/nodejs/node/pull/60916)), where addons reach Node-API through the `napi_env` and link against nothing. That would remove the need for a `node-api` library, but it's still open upstream, with no telling when or whether it lands. Tools like napi-rs would also need to support it before it's a reliable alternative. The JavaScript API and the host in this proposal are needed either way.
- **On by default, with an opt-out.** Compiling Node-API into React Native and its prebuilt artifacts by default would let addons count on it being there, and let apps load them without building from source. It's off by default for now, so apps that don't load addons take on neither the risk nor the binary size while the host is new. It could become the default later (see Unresolved questions).

## Adoption strategy

This isn't a breaking change. Node-API is off unless an app enables it (see Enabling Node-API), so apps and libraries that don't are unaffected, apart from the binary size of prebuilt Hermes under Drawbacks.

Community tooling can move onto it: react-native-node-api's host can be replaced by core's, and its `require()` integration and prebuild tooling can build on `dlopen` and the `node-api` library, without vendoring Hermes.

## How we teach this

- A documentation page for `react-native/node-api`: loading an addon, building one against `node-api`, and how addons behave across reloads.
- The RNTester example, which covers a sync call, async work, a thread-safe function and a reload, as a reference.
- When to choose Node-API over JSI+TurboModules:
  - Node-API: a stable ABI, sharing code with the Node.js ecosystem, prebuilt binaries, and typically one native implementation behind the addon's exports object.
  - TurboModules, and alternatives like Nitro Modules and Expo Modules: tighter integration with React Native, better suited to UI, and typically declared in TypeScript or Flow, with interfaces generated for platform-specific implementations.

## Unresolved questions

- Node lets an addon declare the Node-API version it targets (`node_api_module_get_api_version_v1`, defined by `NAPI_MODULE_INIT`), and keeps older behavior for addons that target older versions. Should Hermes do the same? That's for the Hermes team, and a difference from Node there would affect addons written for Node.
- Should Node-API become the default, or the prebuilt React Native artifacts include it? Apps using them can't load addons today. Turning it on by default would fix that for every app, at the cost of binary size for apps that don't load addons. A variant of the artifacts with it would avoid that cost, at the cost of more artifacts to build and publish.
- Should the implementation include Fantom tests? Fantom runs on the host machine, so a test would need the example addon built for the host as part of Fantom's native build, its first native test fixture. Reload and teardown are covered end to end by the Maestro flow, but only locally: CI builds RNTester without Node-API, and doesn't test React Native's other opt-ins that compile core differently either. Fantom would add faster, more targeted tests of the host, for example of teardown with async work or thread-safe calls still pending.

[^exports]: Unlike Node, the addon's init gets a new, empty exports object rather than the caller's `module.exports`, because Hermes' `hermes_napi_load_module` doesn't take one. Tracked in [meta/hermes#2307](https://github.com/meta/hermes/issues/2307).
[^bionic]: Android's dynamic linker has no case for `DT_FILTER`, and logs it as an "unused DT entry" ([`linker.cpp`](https://android.googlesource.com/platform/bionic/+/refs/heads/main/linker/linker.cpp)).
[^ndkstubgen]: The NDK's sysroot stubs are generated by [`ndkstubgen`](https://android.googlesource.com/platform/build/soong/+/refs/heads/main/cc/ndkstubgen/) from each library's `.map.txt` symbol map.
[^naming]: Like React Native's other opt-ins, the switch is named after the non-default behaviour, and says "enabled": `RCT_NEW_ARCH_ENABLED` and `newArchEnabled` (while the New Architecture was opt-in), `edgeToEdgeEnabled` and `strictCxxApiEnabled`.
[^babel]: For example, react-native-node-api has a Babel transform that resolves a `require` of a Node-API addon to the `filename` passed to `dlopen`.
