---
title: Android - AGP v9 adoption
author:
- Hur Ali
date: 2026-06-09
---

# RFC0000: Android - AGP v9 adoption

## Summary

<!-- Brief explanation of the change. -->

Android Gradle Plugin [v9.0.1](https://developer.android.com/build/releases/agp-9-0-0-release-notes) became stable in January 2026. AGP v9 introduces changes that affect React Native core, community libraries, and consumer applications. AGP v9 provides temporary opt-outs in `gradle.properties`, but those opt-outs will be removed in [AGP v10](https://developer.android.com/build/releases/gradle-plugin-roadmap#agp-10), expected in late 2026.

Key breaking changes include:
- Built-in Kotlin support in AGP v9. Projects should not explicitly apply `"org.jetbrains.kotlin.android"` once this mode is enabled. This is high impact because many libraries and apps currently apply it.
- With built-in Kotlin, `java.srcDirs` cannot be used for Kotlin-only sources. Also, `srcDirs` is deprecated in favor of `.directories.add`.
- `android.kotlinOptions` is removed in favor of `kotlin.compilerOptions`.
- `android.applicationVariants` and `android.libraryVariants` are removed.
- `com.android.build.gradle.LibraryExtension` is deprecated.

This RFC proposes a three-phase AGP v9 adoption strategy:

#### Phase 1
- Enable opt-outs and use the optimized ProGuard file in `react-native-community/template`.
- Upgrade to AGP v9.2.0 and Gradle v9.4.1 in both the template and React Native core.
- Ensure React Native core can build and run with AGP v9.

#### Phase 2
- Roll out AGP v9 compatibility changes to actively used libraries while maintaining backward compatibility.
- Coordinate with the community for broader ecosystem coverage.
- Align with Expo's AGP v9 adoption efforts in Expo core and related libraries.
- Create a new library plugin from RNGP while ensuring backward compat.
  - The RNGP will be restricted to be used only by the App's.
  - This is to allow distributing and maintaining library specific changes independent of RNGP.

#### Phase 3
- Remove the AGP v9 opt-outs from the React Native template.

## Basic example

<!-- If the proposal involves a new or changed API, include a basic code example. Omit this section if it's not applicable. -->

In Phase 1, application `gradle.properties` should include two temporary opt-out flags:

```gradle.properties
# Opt out of built-in kotlin and new DSL behavior that ships with AGP 9.
# Starting from AGP 10.x these opt outs will be removed.
android.builtInKotlin=false
android.newDsl=false
```

In Phase 3, these properties will be removed from `react-native-community/template`.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome?

Please focus on explaining the motivation so that if this RFC is not accepted, the motivation could be used to develop alternative solutions. In other words, enumerate the constraints you are trying to solve without coupling them too closely to the solution you have in mind. -->

Keeping React Native aligned with current AGP versions is required for long-term ecosystem health. Without this upgrade, we risk:

- Consumers being unable to compile Android apps with newer SDK targets, and eventually unable to ship updates when Google Play requires newer target SDK levels. Each AGP version has an upper bound on supported compile/target SDK versions.

- Android Studio build incompatibility. Projects using AGP versions outside the [supported compatibility window](https://developer.android.com/build/releases/about-agp#android_gradle_plugin_and_android_studio_compatibility) may fail to build in newer IDE versions.

- Growing divergence with community Android libraries as they move to AGP v9+, increasing maintenance burden and risking delayed adoption of security fixes.

## Detailed design

<!-- This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here. -->

To measure AGP v9 impact, we ran a discovery phase by enabling AGP v9 on React Native v0.85.3 and validating a set of community libraries. This produced a concrete list of required changes across React Native core and ecosystem packages.

For React Native core, the following changes were required in `react-native-community/template` and `facebook/react-native`:

### react-native-community/template

- Upgrade to Gradle v9.4.1
  - This is already addressed in https://github.com/react-native-community/template/pull/227

- Use optimized ProGuard file
  - This is already addressed in https://github.com/react-native-community/template/pull/223

- Enable opt outs in `gradle.properties`
  - This is already addressed in https://github.com/react-native-community/template/pull/224

### facebook/react-native

- Upgrade to Gradle v9.4.1
  - This is already addressed in https://github.com/facebook/react-native/pull/57078

- Adopt AGP v9
  - Upgrade to AGP v9.2.0
  - Remove `alias(libs.plugins.kotlin.android)` in favor of built-in Kotlin
  - Add Kotlin-only sources to `kotlin.srcDirs` instead of `java.srcDirs`
  - Replace `.srcDirs` with `.directories`
  - Fix RNGP behavior for AGP v9 strictness around invoking `finalizeDsl` after the phase has passed
  - All of the above changes are addressed in https://github.com/facebook/react-native/pull/57038

This completes Phase 1 described above.

<hr/>

The second area is AGP v9 compatibility for community libraries. Below are the common migration patterns and library-specific notes.

### Community libraries

Below is the matrix used during discovery:
  
| Library | Version | Patch link |
| --- | --- | --- |
| react-native-reanimated | v4.3.1 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-reanimated%2B4.4.1.patch |
| react-native-worklets | v0.8.3 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-worklets%2B0.9.1.patch |
| react-native-screens | v4.25.2 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-screens%2B4.25.2.patch |
| react-navigation | no issues | N/A |
| react-native-safe-area-context | v5.7.0 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-safe-area-context%2B5.7.0.patch |
| react-native-vision-camera | v5.0.10 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-vision-camera%2B5.0.10.patch |
| react-native-skia | no issues - v2.6.3 | N/A |
| react-native-pager-view | v8.0.2 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-pager-view%2B8.0.2.patch |
| react-native-nitro-modules | v0.35.7 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-nitro-modules%2B0.35.7.patch |
| react-native-svg | no issues - v15.15.5 - Pure java | N/A |
| react-native-mmkv | v4.3.1 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-mmkv%2B4.3.1.patch |
| react-native-share | no issues - v12.3.1 | N/A |
| async-storage | v3.1.0 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/%40react-native-async-storage%2Basync-storage%2B3.1.0.patch |
| nitro-image | v0.14.0 | https://github.com/hurali97/RN-AGP9-support/blob/main/patches/react-native-nitro-image%2B0.14.0.patch |

<hr/>

- Most libraries apply the `kotlin-android` plugin in `build.gradle`. With built-in Kotlin, this is no longer needed and can fail builds.
- Previously, `java.srcDirs` plus `kotlin-android` covered Kotlin-only sources. With built-in Kotlin, Kotlin-only sources should be defined via `kotlin.srcDirs`.
- `.srcDirs` is deprecated in favor of `.directories`.
- `android.kotlinOptions` is no longer valid with built-in Kotlin; use `kotlin.compilerOptions`.
- `fix-prefab.gradle` scripts in C++ (CMake) libraries often use removed APIs (`android.applicationVariants` / `android.libraryVariants`); migrate to `androidComponents.onVariants`.

The points above cover common migration work. One notable library-specific case:

- The `nitrogen` template (used by Nitro modules) defines `java.srcDirs` for Kotlin-only sources. With built-in Kotlin this must move to `kotlin.directories.add`.

</details>

<hr/>

**Expo**:
[TBA]

<hr/>

### Separate Library Plugin

Currently, the react-native applications and libraries are both handled by the RNGP(`com.facebook.react`) to apply auto-linking, codegen changes, etc. This means that the app and library specific handling is fused within the RNGP and doing a major change only for library means we ensure the app handling is not breaking, so to avoid this friction, we now create a new library gradle plugin by extracting pieces from RNGP and putting it together as `com.facebook.react.library`.

**Details**
- Introduce a new module within `gradle-plugins` and register it, so it can be used via comosite build
  - Since it will be used via composite build, we do not expect any change in user's `settings.gradle`
- Extract the library pieces out to a separate gradle plugin
  - This means if there are any code in RNGP that the library plugin needs, we duplicate that implementation.
  - This is a temporary trade-off until we fully deprecate RNGP for librares, see below.
- Add deprecation notice to RNGP to warn the users over a new couple releases to start using dedicated `com.facebook.react.library` plugin.
  - Upon reaching the deprecation window, we remove the library specific code from RNGP.

> A draft version of this can be found here - https://github.com/react/react-native/pull/57912

**Community**
- This will require us to update `react-native-builder-bob` to use `com.facebook.react.library` by default.

<hr/>

## Adoption strategy

<!-- If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries? -->

As described in the earlier section, AGP v9 adoption is rolled out in three phases. Phase 1 is complete and should ship with React Native v0.87.x. Newly created apps do not need changes because template opt-outs are enabled by default. Existing apps upgrading to v0.87.x should apply a few manual updates described in the next section. At this stage, we expect no breakages with actively maintained community libraries.

In Phase 2, we will submit compatibility pull requests to actively used and maintained libraries from the matrix above while preserving backward compatibility. Broader ecosystem coverage will require community participation.

As part of Phase 2, the Expo team will evaluate required changes on their side, including ecosystem impact.

The separate library gradle plugin will also be landing as part of the Phase 2.

Phase 2 needs sufficient time, especially for community adoption.

Once there is enough confidence and ecosystem coverage, we can move to Phase 3: remove opt-outs from `react-native-community/template` and ask existing apps to remove them as well. At that point, the ecosystem should fully leverage built-in Kotlin, new DSL behavior, and other AGP v9 defaults.

## How we teach this

<!-- What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers? -->

In Phase 1, instructions are needed only for existing applications. New applications should work out of the box. We can add a dedicated AGP v9 section in the React Native v0.87.x release post with upgrade guidance.

For existing apps:

- Upgrade the application to react-native v0.87.x
- Define the `kotlinVersion = "2.2.0"` in `android/build.gradle`
- Bump Gradle to v9.4.1 in `android/gradle/gradle-wrapper.properties`
- Add the following opt outs to `android/gradle.properties`
```gradle.properties
# Opt out of built-in kotlin and new DSL behavior that ships with AGP 9.
# Starting from AGP 10.x these opt outs will be removed.
android.builtInKotlin=false
android.newDsl=false
```

In addition to manual instructions, we should update `react-native-community/upgrade-helper` so this guidance is included by default.

In Phase 2, we will migrate widely used libraries. To accelerate this, we should include a public request for help in the v0.x.x release post and X announcements.

As part of the release post, we should add section about deprecating library support from RNGP and recommend using `com.facebook.react.library`. The section should include:

- This is a breaking change for libraries.
- If this change is shipped, say in 0.89.0 release, the apps when they upgrade to this version will see a deprecation notice for each third party library not using `com.facebook.react.library`. This allows the community a decent window to start adopting the new library gradle plugin.
- In the next few minor releases, the RNGP support for libraries will be removed completely.
- We will backport the change to the react-native versions in the release window.
- The react-native versions outside of the release window, will not work with the new library plugin.
  - The third party libraries may need to allow their users to use a version of their library which is compatible with the react-native version below the lower bound of release window. (For eg, 0.85.0 is the lower bound for 0.87.0 main release)

For Phase 3, we should announce opt-out removal from `react-native-community/template` in the v0.88.x release post and state the expected removal timeline (for example, v0.89.x).

## Unresolved questions

<!-- Optional, but suggested for first drafts. What parts of the design are still TBD? -->

- Expo needs to evaluate its scope of AGP v9 changes and provide feedback on adoption timelines and risks.
- When the RNGP support for libraries will be removed completely? (Ideally, a time frame.)
