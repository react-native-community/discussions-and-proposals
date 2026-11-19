---
title: Platform-agnostic React Native core
author:
  - Peter Abbondanzo
  - Cedric van Putten
date: 2026-11-19
---

# RFC0000: Platform-agnostic React Native core

## Summary

React Native core should only contain shared C++ code. Today, iOS and Android are wired directly into it: their implementations live inside ReactCommon, they're picked by build logic that only knows about those two platforms, and shared code checks which of them it's being compiled for. Every other platform (macOS, Windows, Vega, HarmonyOS, tvOS) has to patch our files to plug in, then spend weeks re-merging those patches every release.

**We propose splitting the `react-native` package into `react-native-core`, `react-native-ios`, and `react-native-android`, with core defining one clear interface that every platform implements, iOS and Android included.** Core declares what a platform has to provide but implements none of it. The existing `react-native` package composes the three and re-exports them, so it looks and behaves exactly the same to existing users. **We are _not_ proposing that we move iOS or Android out of the monorepo or change any public API (for now).**

## Basic example

An out-of-tree platform becomes a package with the same shape as `react-native-ios` or `react-native-android`. It declares itself in its `package.json`:

```json
{
  "name": "react-native-macos",
  "reactNative": {
    "platform": {
      "name": "macos",
      "extends": "react-native-ios",
      "variant": "desktop",
      "hostInterface": ">=1 <2",
      "hostPlatform": "./hostplatform",
      "js": "./js",
      "codegen": "./codegen/index.js"
    }
  }
}
```

It then provides the types core needs from one header, reusing iOS wherever macOS doesn't differ:

```cpp
// react-native-macos/hostplatform/react/hostplatform/HostPlatformBindings.h
#include <react/hostplatform/ios/IOSHostTypes.h>

namespace facebook::react::hostplatform {
using ViewProps = MacOSViewProps;
using ViewEventEmitter = MacOSViewEventEmitter;
using Color = ios::Color;
// ...
} // namespace facebook::react::hostplatform
```

Nothing in core changes to add macOS. If macOS forgets something core needs, the build fails with an error that names it.

## Motivation

ReactCommon is supposed to be our shared, cross-platform C++, but in practice a lot of platform code lives there too. As of writing, ReactCommon has 82 files under `platform/ios/` folders, 49 under `platform/android/`, and 62 Objective-C++ files. That includes the entire Apple bridgeless host (`RCTHost`, `RCTInstance`) and the Objective-C TurboModule bridge. Outside of ReactCommon, `Libraries/` mixes our JS with 73 Objective-C files and 14 podspecs.

The way we pick a platform doesn't help. Core includes about 20 headers it doesn't own, like `HostPlatformViewProps.h`, and relies on whichever `platform/<os>/` folder comes first on the include path to provide them. That selection is written three separate times: CMake's `react_native_android_selector` (which is literally "Android, or else the fallback"), hardcoded `platform/ios` header search paths in the podspecs, and, at Meta, a BUCK `select()` with one arm per platform. Nothing lists these headers or checks that a platform provides them, so a platform that forgets one quietly gets the fallback. Shared code also checks the platform directly. 55 files gate behavior on `RN_SERIALIZABLE_STATE`, which only Android defines, and more than 30 others check `ANDROID`.

JS and codegen have the same problem. Out-of-tree platforms plug into Metro by redirecting every `react-native` import to their own package, which only works if that package carries a full copy of our JS. Shared JS has 267 `Platform.OS` checks against `'ios'` or `'android'` and 25 `.ios.js`/`.android.js` files, and codegen hardcodes `supportedPlatforms = ['android', 'ios']`.

Put all of this together and there's nowhere for an out-of-tree platform to put its code except inside our files. react-native-macos carries its macOS support as patches inside our `platform/ios/` files, and react-native-tvos maintains a full fork of the repo. These teams spend months on merge conflicts that a clear boundary would turn into compile errors.

## Detailed design

Core gets a single folder, `react/hostplatform/`, that declares everything a platform needs to provide, and platforms implement it from their own packages. iOS and Android go through the exact same interface as any out-of-tree platform. If either of them needs something new from core, it goes into the interface and every platform gets it.

Two rules keep the boundary clean. First, core contains shared C++ and interfaces only: no Objective-C, Java, Kotlin, or JNI, and no default implementations of anything a platform provides. Second, core can branch on the OS or compiler for portability (`oscompat/`, the Windows export macros in `jsi.h`), but never on which React Native platform it's being built for.

Today's `platform/cxx` fallbacks don't go away. They move into a fourth package, `react-native-cxx`, along with the pure C++ host from `ReactCxxPlatform/` that Fantom already runs on. That becomes the reference platform, and it's what core's own tests build against.

### Package layout

- `packages/`
  - `react-native/`
    - `index.js`, `index.js.flow`, `ReactNativeApi.d.ts` Public JS API + composition
    - `Libraries/`, `src/` Shared JS primitives (Phase 2 moves platform JS out)
    - `scripts/react_native_pods.rb` Shim to react-native-ios
    - `package.json` Depends on core/ios/android at the exact same version
  - `react-native-core/` Shared C++ interfaces
    - `ReactCommon/` Today's ReactCommon without the `platform/<os>/` dirs
      - `react/hostplatform/` The interface!
        - `HostPlatform.h`
        - `Concepts.h`
        - `Functions.h`
        - `services/` Abstract service interfaces
        - `CONTRACT.md` Human-readable list of every point
        - `contract.json` Machine-readable version of the above
    - `ReactCxxPlatform/` Portable modules that only depend on interfaces
    - `conformance/` Fantom!
    - `cmake/` `react-native-core-config.cmake`
    - `CMakeLists.txt` Canonical, platform-neutral build
    - `package.json`
  - `react-native-cxx/` Reference platform: pure C++, headless
    - `hostplatform/react/hostplatform/HostPlatformBindings.h`
    - `hostplatform/` Today's `platform/cxx` implementations
    - `host/` ReactHost, threading, HTTP/WebSocket, resource loading
    - `package.json` Platform manifest (`reactNative.platform`)
  - `react-native-ios/` Apple platform
    - `hostplatform/` Bindings + iOS implementations of host points
    - `React/`, `ReactApple/` From react-native/ (+ Apple-only code from ReactCommon)
    - `Libraries/` Native iOS modules from `react-native/Libraries/*`
    - `js/` (Phase 2) iOS implementations of JS host points
    - `codegen/` (Phase 2) Objective-C++ generators
    - `*.podspec`, `Package.swift`, `third-party-podspecs/`, `scripts/`
    - `package.json` Platform manifest (`reactNative.platform`)
  - `react-native-android/` Android platform
    - `hostplatform/` Bindings + Android implementations of host points
    - `ReactAndroid/` As today: `src/main/java`, `src/main/jni`, `cmake-utils`
    - `js/` (Phase 2) Android implementations of JS host points
    - `codegen/` (Phase 2) Java/JNI generators
    - `build.gradle.kts`, `settings.gradle.kts`, `gradle/`
    - `package.json` Platform manifest (`reactNative.platform`)

An out-of-tree platform has the same shape as `react-native-ios` or `react-native-android`: a `hostplatform/` directory with its bindings and implementations, its own native code, optional `js/` and `codegen/` directories, and a manifest entry in `package.json`.

### What moves where

Shared headers keep their include paths, so `#include <react/renderer/...>` doesn't change anywhere. Only the package root moves.

| Today | Moves to |
| --- | --- |
| `ReactCommon/**` (shared C++) | `react-native-core/ReactCommon/**` |
| `ReactCommon/**/platform/cxx/**` | `react-native-cxx/hostplatform/**` |
| `ReactCommon/**/platform/ios/**` | `react-native-ios/hostplatform/**` |
| `ReactCommon/**/platform/android/**` | `react-native-android/hostplatform/**` |
| `ReactCommon/react/runtime/platform/ios/**` (`RCTHost`, `RCTInstance`) | `react-native-ios/ReactApple/Runtime/**` |
| `ReactCommon/react/nativemodule/core/platform/ios/**` | `react-native-ios/ReactApple/NativeModule/**` |
| `ReactCommon/react/nativemodule/core/platform/android/**` | `react-native-android/ReactAndroid/src/main/jni/**` |
| `ReactCxxPlatform/` interfaces and portable modules | `react-native-core/` |
| `ReactCxxPlatform/` concrete host (`ReactHost`, threading, `platform/cxx`) | `react-native-cxx/host/` |
| `ReactCxxPlatform/react/jni/`, `io/platform/android/` | `react-native-android/` |
| `React/`, `ReactApple/`, native code in `Libraries/*/` | `react-native-ios/` |
| `ReactAndroid/`, Gradle files | `react-native-android/` |

### The host platform interface

Core reaches a platform in three ways.

**Types.** Some platform types sit on hot paths like props parsing and shadow nodes, so they need to stay compile-time. Instead of shadowing ~20 headers, core includes exactly one header the platform owns, `HostPlatformBindings.h`, and checks it with C++20 concepts:

```cpp
// react-native-android/hostplatform/react/hostplatform/HostPlatformBindings.h
namespace facebook::react::hostplatform {
using ViewProps = AndroidViewProps;
using ViewEventEmitter = BaseViewEventEmitter;
using ParagraphState = AndroidParagraphState;
using Color = AndroidColor;
// ...one alias per type point
} // namespace facebook::react::hostplatform

#define RN_HOST_SERIALIZABLE_STATE 1

// react-native-core/.../react/hostplatform/HostPlatform.h
#include <react/hostplatform/HostPlatformBindings.h>
#include <react/hostplatform/Concepts.h>

static_assert(HostViewProps<hostplatform::ViewProps>,
    "ViewProps must derive from BaseViewProps");
```

If a platform's `ViewProps` doesn't derive from `BaseViewProps`, or is missing something core needs, it fails to compile with a message that names the type. A platform that's happy with core's base types just aliases them, which is effectively what iOS does for view props today.

**Functions.** Small utilities like `parsePlatformColor`, `LowPriorityExecutor`, and `RuntimeExecutorSyncUIThreadUtils` are declared in core and defined by the platform. A missing one is a link error.

**Services.** Bigger subsystems like `TextLayoutManager`, `ImageManager`, and networking become abstract interfaces that the platform registers at startup through `ContextContainer`. These calls already cost far more than a virtual dispatch, so there's no meaningful overhead.

Shared code that checks for Android today (`RN_SERIALIZABLE_STATE`, `ANDROID`) checks capability flags from the bindings header instead, and a lint keeps new platform checks out of core. TextInput also needs a shared base. iOS and Android currently define two unrelated components (`iostextinput` and `androidtextinput`), so core gains a `TextInputProps`/`TextInputState` pair that both extend.

Platforms can build on each other, but they have to say so, like react-native-macos does in the [basic example](#basic-example): it includes iOS's types in its bindings header, overrides the handful that differ, and registers iOS's services wherever they work as-is.

Every point is listed in `CONTRACT.md` and `contract.json` along with the interface version it was added in. Core defines `RN_HOST_PLATFORM_INTERFACE_VERSION`, platforms declare which versions they support, and changing or removing a point requires a release where both versions exist. Conformance tests run on Fantom, and every platform runs them in CI.

One thing worth calling out: because types are compile-time, core is always built against a platform. There's no platform-independent core binary, but that's true today too.

### Build integration

Each build system gets one hook in core, and every platform uses it the same way. In CMake, a platform exports a `ReactNative::HostPlatform` target that provides its bindings header and implementation library, and core is configured against it. That replaces `react_native_android_selector`. Gradle passes Android's target into the same CMake, so the published `react-android` artifact doesn't change. CocoaPods and SwiftPM get the bindings folder from react-native-ios instead of per-component `platform/ios` search paths, and at Meta, BUCK targets depend on one `hostplatform` alias instead of a `select()` per platform.

CocoaPods and SwiftPM have one wrinkle: sources have to live under the pod or package root, so the 52 podspecs that build core's C++ can't just move to react-native-ios. They stay with core's sources during the migration, pulling only the bindings folder from iOS, and leave core once iOS builds it through the prebuilt path.

### Platform manifest

Each platform declares itself in its `package.json`. The CLI already finds out-of-tree platforms through `npmPackageName`, so this extends that idea until the CLI, Metro, codegen, and native builds all read the same thing instead of hardcoding `['android', 'ios']`:

```json
{
  "name": "react-native-ios",
  "reactNative": {
    "platform": {
      "name": "ios",
      "extends": null,
      "variant": "mobile",
      "hostInterface": ">=1 <2",
      "hostPlatform": "./hostplatform",
      "cmake": "./cmake/ReactNativeIOS.cmake",
      "js": "./js",
      "codegen": "./codegen/index.js"
    }
  }
}
```

`name` is the platform's `Platform.OS` value and `variant` is its `Platform.Variant` (more on both in [JS and codegen](#js-and-codegen-phase-2) below). react-native-macos would set `"extends": "react-native-ios"` and `"variant": "desktop"`, which lets it inherit iOS's JS modules and codegen while implementing the `Desktop` APIs.

### Keeping `react-native` the same

`react-native` depends on core, iOS, and Android at its own exact version, and they all release in lockstep from the monorepo like our `@react-native/*` packages do today. JS imports don't change. `react_native_pods.rb` becomes a shim that forwards to react-native-ios, the Gradle plugin keeps `reactNativeDir` working, and pod names and Maven coordinates stay the same.

The hard part is libraries. Plenty of them hardcode `node_modules/react-native/ReactCommon` or `node_modules/react-native/ReactAndroid` in their podspecs and CMake files. For a few releases, the `react-native` tarball will include those folders at their old paths, generated at publish time, while `react-native doctor` and codegen warn about the old paths and we publish helpers that resolve the new ones.

### JS and codegen (Phase 2)

This comes after the native split and follows the same idea. Shared JS imports platform behavior only from `react-native/hostplatform/<Module>`, and the CLI's Metro config resolves that to the platform package's `js/` folder (or the platform it extends). If nothing provides the module, the build fails and names it. This is a generalization of the existing `reactNativePlatformResolver`, which today redirects _all_ of `react-native` to the platform package. Out-of-tree platforms can then stop shipping a copy of our JS, and react-native-windows would ship `js/Image.js` and `js/Platform.js` instead of a fork of `Libraries/`.

Doug Lowder's [React Native OOT platform API](https://github.com/react-native-community/discussions-and-proposals/discussions/1009) discussion (#1009) lays out the JS side of this really well, and we'd like this phase to adopt it. Each platform owns its `Platform.OS` value, declared as `name` in its manifest, and that one value feeds the runtime `Platform` module, its native APIs, the bundle URL it sends to Metro, and the build step that embeds its bundle. That means `Platform.OS` stops being a closed list in core. `native` can go away, since it's really just Metro's way of saying "not web", and values like `tvos`, `androidtv`, `vega`, and `visionos` come from the platforms that ship them.

The discussion also proposes `Platform.Variant` (`mobile`, `desktop`, `tv`, `headset`, `web`), which describes the form factor instead of the OS. It sits nicely next to `extends` in the manifest: `extends` says whose code you build on, and `variant` says which family of APIs you implement. APIs that only make sense for one family move into a namespace for that variant, like `TV.FocusGuideView` and `TV.EventHandler`, so react-native-tvos and Vega implement the same `TV` API and react-native-macos and react-native-windows implement the same `Desktop` API. Core JS declares those namespaces and each platform provides them as JS host points. The resolver checks the platform first, then the platforms it extends, then code shared by its variant, so `androidtv` resolves through `androidtv`, `android`, and then `tv`, the same order #1009 suggests for file extensions. Checks in shared JS that are really about form factor, like the 18 uses of `Platform.isTV` and `Platform.isVision`, become `Platform.Variant` checks. And since Metro only pulls in the active platform's `js/` folder, a TV bundle never picks up desktop code, which covers the bundle size concern raised in the discussion.

The 25 `.ios.js` and `.android.js` files become those modules and move into the iOS and Android packages. APIs like `ToastAndroid` and `ActionSheetIOS` stay exported from `react-native`, with a shared "unavailable" stub for platforms that don't provide them. `Platform.OS` checks in shared JS turn into capability checks or move behind a module, every `Platform.select` gets a `default`, and a lint rule blocks new platform checks.

Codegen splits by output language. The schema, parsers, and the C++ and JS generators stay in `react-native-codegen`, while the Objective-C++ generators move to react-native-ios and the Java/JNI generators move to react-native-android. Platforms register their generators through the manifest, codegen discovers platforms from installed manifests instead of `supportedPlatforms`, and macOS reuses iOS's generators through `extends`.

## Plan of action

The work is split into three phases, each with its own owners. None of the steps break apps, and each one can ship on its own.

### Phase 1: Modularize core

- **Owner:** Cedric van Putten (Expo)
- **Co-owner:** Peter Abbondanzo (Meta)
- **Members:** Expo, Callstack, Platform Expertise working group
- **Size:** M (1–4 person-quarters)
- **Depends on:** Lean Core 2.0 (Stable API), the event loop work (Runtime)

Phase 1 is done when out-of-tree platforms can integrate through published platform extension APIs and a written integration contract, instead of patching core. We get there in small milestones, and none of them move a file until the build already knows how to find it in both places.

1. **Declare the interface in place.** Add `react/hostplatform/` with the contract, its validator, and the concepts. Give every existing platform a bindings header, route each point through a checked seam, and replace `RN_SERIALIZABLE_STATE` and the `ANDROID` behavior checks with capability flags. Add the lint and CI job that keep new platform checks out of core. No files move. This is done when platform headers are only included through checked seams.
2. **One platform selector.** Replace `react_native_android_selector`, the hardcoded podspec search paths, and the per-platform `select()` arms with a single registry outside ReactCommon that drives every build system. Convert one library at a time and check that each conversion leaves compile commands unchanged. This is done when adding a platform doesn't touch anything in ReactCommon.
3. **Placeholder packages and CI.** Add `react-native-core`, `react-native-cxx`, `react-native-ios`, and `react-native-android` as private, unpublished workspace packages, plus a CI job that builds core against `react-native-cxx` and runs Fantom. Teach build files to accept either the old or the new location for a file, and start tracking moves in `MOVED_PATHS.json`. We're deliberately not bridging old and new paths with symlinks: they aren't reliable on Windows checkouts or in published npm tarballs, and dual-location build files get us the same result.
4. **First move.** Move a single file, modal's `platform/cxx/ModalHostViewUtils.cpp`, into `react-native-cxx` as a pure rename, and confirm that `git log --follow` and blame follow it. This proves the history-preserving path before anything bigger moves.
5. **Move the rest, one folder at a time.** Each remaining `platform/<os>/` folder moves in its own pure-rename commit: cxx first, then iOS, then Android. After that come `React/`, `ReactApple/`, and the native code in `Libraries/`, then `ReactAndroid/`, then the `ReactCxxPlatform/` split. Content changes never ride along with a rename, so git's rename detection keeps history intact for everyone rebasing on top. This is done when core has no Objective-C, Java, or Kotlin and its CI is green against `react-native-cxx`.
6. **Publish and compose.** Settle package names, publish the packages, switch `react-native` over to depending on them, and ship the compatibility folders. Before this lands, popular libraries that build against core (Expo and Reanimated both read `RN_SERIALIZABLE_STATE` today) have to build unchanged.
7. **Publish interface v1.** Freeze `CONTRACT.md` and `contract.json` at v1 and publish the integration contract and participation rules for out-of-tree platforms.

### Phase 2: Reconcile cross-platform APIs

- **Owner:** Andrew Datsenko (Meta)
- **Co-owner:** Anisha (Amazon)
- **Members:** TBD
- **Size:** M (1–4 person-quarters)

Phase 2 is done when a reference out-of-tree platform passes public keyboard, focus, library, and integration tests against the shared contract.

1. **Pick the reference platform** and assign owners for keyboard and focus.
2. **Publish the test plans** for keyboard, focus, and libraries, and extend `conformance/` to cover JS.
3. **JS host points.** Add the `react-native/hostplatform/<Module>` resolver, move the 25 platform-suffixed files into the platform packages, and add `Platform.select` defaults and the lint, as described in [JS and codegen](#js-and-codegen-phase-2).
4. **`Platform.OS` and `Platform.Variant`.** Let platforms own their `Platform.OS` value, add `Platform.Variant`, and move form-factor APIs into variant namespaces, following #1009.
5. **Codegen.** Move the Objective-C++ and Java/JNI generators into their platform packages and switch codegen and Metro to manifest discovery.

This is done when `Libraries/` has no platform-suffixed files or platform checks, codegen has no hardcoded platform list, and the reference platform ships no copy of our JS.

### Phase 3: Platform implementations

- **Owner:** Cedric van Putten (Expo)
- **Co-owner:** Peter Abbondanzo (Meta)
- **Members:** All companies and working groups
- **Size:** L (more than 4 person-quarters)

Phase 3 is where platforms actually adopt the interface and we refine it based on what they hit.

1. **Platforms move onto the interface.** macOS on top of iOS, plus Windows, tvOS, visionOS, Vega, HarmonyOS, and others. Each one stops overriding files in core and stops shipping a copy of our JS.
2. **Share what can be shared.** Where platforms in the same variant can share code (a `TV` implementation for both tvOS and Vega, for example), it moves somewhere they can both depend on it.
3. **Stabilize the interface.** Refine the host points, hooks, props, and components based on adoption until they're well defined, then mark them stable.
4. **Close the compatibility window.** Remove the compatibility folders and core's podspecs.

This is done when out-of-tree platforms no longer override anything in core's tree and the platform APIs are stable.

## Drawbacks

The biggest cost is a one-time move. Every out-of-tree fork and open PR will need to rebase once. We can soften that by moving one area at a time on announced dates, keeping shared include paths the same, publishing a `MOVED_PATHS.json`, and using `git mv` so history follows the files.

Dropping defaults means every platform, iOS and Android included, has to spell out every type in its bindings even when it just aliases the base type. That's more boilerplate, but it's kind of the point: forgetting something should be an error. The interface also becomes API, so changing a type means a version bump and a deprecation window, which will slow down some internal refactors.

## Alternatives

The closest alternative is keeping defaults in core and letting platforms override them, which is basically today's model written down. It's less work, but platform code stays in core and a missed override silently falls back instead of failing. Templating core on a platform traits type would give us an even stronger boundary, but it's very invasive and would slow compile times considerably. Making everything a runtime interface is cleaner on paper but adds virtual calls and allocations on the props and shadow node hot paths. Moving iOS and Android into their own repos loses atomic cross-platform changes without making the layout any better than separate packages in one repo.

We could also rename `ReactCommon/` while we're at it. We'd rather not combine a move and a rename since it makes rebasing harder for everyone, and renaming inside core later is cheap.

### Prior art

react-native-windows already keeps its Fabric platform files outside core under `Fabric/platform/react/renderer/...`, and this gives that pattern a real interface instead of a shadowed include path. `ReactCxxPlatform/` already separates interfaces (`IHttpClient`, `IImageLoader`) from implementations and powers Fantom. Extracting iOS and Android into their own packages was proposed back in 2019 (#496), and the out-of-tree platforms session at the 2024 Core Contributor Summit landed on reducing core's surface area. More recently, the [React Native OOT platform API](https://github.com/react-native-community/discussions-and-proposals/discussions/1009) discussion started working through `Platform.OS` values, a `Platform.Variant` API, and Metro resolution for out-of-tree platforms, and Phase 2 here builds directly on it.

## Adoption strategy

For app developers, nothing changes. `react-native` keeps its name, its JS API, its pod names, and its Maven coordinates, and it pulls in the new packages at the same version.

Library authors who reference `node_modules/react-native/ReactCommon` or `node_modules/react-native/ReactAndroid` keep working through the compatibility folders for a few releases. `react-native doctor` and codegen warn about the old paths, and we publish Ruby, Gradle, and CMake helpers that resolve the new ones.

Out-of-tree platforms migrate one host point at a time. Each point is versioned in `contract.json`, and a point only changes or goes away after a release where both versions are available.

## How we teach this

`CONTRACT.md` doubles as the porting guide, with react-native-cxx as the worked example. On top of that, we'd add a "Build a platform" page on reactnative.dev, a migration note for library authors on the new path helpers, and release notes for each move that link to `MOVED_PATHS.json`.

## Unresolved questions

1. Should the packages be unscoped (`react-native-ios`, peers of `react-native-windows`) or scoped (`@react-native/ios`, like the rest of the monorepo)? Unscoped names also need an npm availability check.
2. Should shared JS stay in `react-native`, or move into core (or its own package) so `react-native` only composes the others?
3. How long do we keep the compatibility folders around: two releases, or until the next major?
4. Is one release of overlap enough for out-of-tree platforms that trail upstream by several releases?
5. Where do Hermes's Apple and Android build pieces in `sdks/` end up?
6. Should tvOS and visionOS build on react-native-ios the same way macOS does, so react-native-tvos no longer needs a full fork?
7. Where does JS shared by a whole variant live, for example a `TV` implementation that works for both tvOS and Vega: in `react-native`, or in a variant package like `react-native-tv`?

_File counts are approximate and were taken from `packages/react-native` in November 2026._
