---
title: Swift Package Manager Support for React Native iOS Projects
author:
  - Christian Falch (@chrfalch)
date: 2026-04-09
---

# RFC0000: Swift Package Manager Support for React Native iOS Projects

## Summary

This RFC proposes adding first-class support for Swift Package Manager (SPM) for React Native on Apple platforms, with the goal of replacing CocoaPods over a staged transition period (see Cocoapods blogpost here: CocoaPods Trunk Read-only Plan).

With CocoaPods deprecation targeted for December 2026, React Native needs a supported SPM-based integration story for React Native core, Hermes, Codegen, autolinking, and third-party native libraries. Cocoapods will be read-only and prevent creation of new libraries.

## Motivation

React Native’s iOS integration is currently tightly coupled to CocoaPods. This creates several issues:

- React Native apps require a Ruby/CocoaPods toolchain in addition to the standard Xcode toolchain.
- Some Apple ecosystem dependencies are now distributed primarily through Swift Package Manager. They can still be used in the current setup, but with some limitations.
- React Native library authors must maintain CocoaPods integration even when SPM is the preferred native packaging format.
- The current setup diverges from the default dependency workflow used by many Apple developers.

If React Native intends to deprecate CocoaPods in December 2026, it needs a migration path that is practical for app developers, library maintainers, and the CLI/tooling ecosystem. The motivation for this RFC is therefore not just to add another packaging option, but to define a credible path from today’s CocoaPods-based workflow to a supported SPM-based workflow.

## Basic example

NOTE: The following examples use the demo branch but will be implemented in react-native-cli with the same parameters and interface.

For a new project, the prototype SPM setup is initialized with:

```bash
yarn ios --init # it calls node node_modules/react-native/scripts/setup-ios-spm.js --init under the hoods

```

That command:

- runs React Native Codegen,
- generates `autolinked/Package.swift`,
- downloads the required prebuilt XCFramework artifacts,
- generates the local SPM packages consumed by the app, and
- creates an Xcode project configured for the SPM flow.

After initialization, manual refresh is done with:

```bash
yarn ios # it calls node node_modules/react-native/scripts/setup-ios-spm.js under the hoods
```

And a full reset of generated state is done with:

```bash
yarn ios --clean # it calls node node_modules/react-native/scripts/setup-ios-spm.js --clean under the hoods

```

## Detailed design

React Native should support two iOS integration flows during the migration period:

- CocoaPods, as the existing integration path.
- Swift Package Manager, as the new first-class integration path.

### React Native core packaging

The current prototype already implements a working SPM packaging model in the chrfalch/swift-package-manager branch.

At a high level, React Native core is exposed to app projects as a small set of Swift packages centered around prebuilt XCFrameworks:

- `ReactNative`, which packages the React Native iOS runtime
- `ReactNativeDependencies`, which packages the native third-party dependencies used by React Native
- `hermes-engine`, which provides the Hermes runtime

The prototype uses a single script, `setup-ios-spm.js`, to prepare an app for this layout. That script:
runs Codegen,
generates SPM autolinking,
downloads the required artifacts,
generates the package manifests
creates an Xcode project configured for the SPM flow.

From the app’s point of view, the integration stays relatively simple: the app’s `Package.swift` depends on three locally generated packages for React Native core, generated code, and autolinked native modules. The generated Xcode project also includes an auto-sync build phase that refreshes autolinking when dependencies change.

This RFC does not propose a new packaging model from scratch. Instead, it proposes standardizing and evolving the implementation direction already demonstrated in the prototype branch.

### Autolinking

The current prototype keeps the React Native autolinking model, but changes the output from CocoaPods configuration to Swift Package Manager configuration.

Instead of generating Podfile integration, the SPM flow generates a local `autolinked/Package.swift` that collects the native React Native dependencies discovered in the app. This package is then added as a dependency of the app’s main `Package.swift`, alongside the generated Codegen package and the React Native core package.

At a high level, the flow is:

1. React Native Codegen generates the native metadata and codegen artifacts for the app.
2. The SPM autolinking step reads the generated autolinking metadata and the app’s `react-native.config.js`.
3. It produces `autolinked/Package.swift`, which contains local package targets for the app’s native dependencies.
4. The app’s generated Xcode project depends on that package, so native modules are available through normal SPM package resolution.

This keeps the developer-facing model close to today’s React Native workflow: dependencies are still discovered from the JavaScript project, but the integration backend is SPM instead of CocoaPods.
Local modules
The prototype also supports additional local modules through `spmModules` in `react-native.config.js`. This makes it possible to include native code that is not otherwise discovered through standard autolinking.

For example:

```js
module.exports = {
  spmModules: [
    {
      name: "MyNativeModule",
      path: "ios/MyNativeModule",
      exclude: ["*.podspec"],
      publicHeadersPath: ".",
    },
  ],
};
```

For a new project, the initial setup is performed with:

```bash
yarn ios --init
```

This first-time setup generates the initial `Package.swift`, the generated SPM packages, and the Xcode project for the app.

After initialization, autolinking is normally kept up to date automatically by the generated Xcode project. The prototype adds an auto-sync build phase that detects dependency changes and regenerates the autolinked package before compilation when needed.

When a manual refresh is needed, for example after changing native dependencies or local module configuration, developers can re-run:

```bash
yarn ios
```

If generated state needs to be rebuilt from scratch, the prototype also supports:

```bash
yarn ios --clean
```

### Codegen

The current prototype keeps React Native Codegen as part of the normal iOS setup flow, but packages its output for Swift Package Manager instead of CocoaPods.

In the implementation branch, `setup-ios-spm.js` starts by running the existing React Native Codegen pipeline and writing the generated output to `build/generated/ios/`. After that, it installs an SPM-specific `Package.swift` into that generated directory so the output can be consumed as a local Swift package.

From the app’s point of view, the generated code is exposed through a separate package, `React-GeneratedCode`, with products such as `ReactCodegen` and `ReactAppDependencyProvider`. The app’s main `Package.swift` then depends on that generated package alongside the React Native core package and the autolinked package.

This keeps Codegen in roughly the same place in the developer workflow as today: it still runs before native compilation, but its output is expressed as SPM package targets rather than Pod targets.

We’ll also port the current detection of changes in the Spec files to the new SPM packaging system.

### Hermes and native dependencies

The current prototype already includes a concrete packaging model for Hermes and React Native’s native third-party dependencies.

Instead of asking each app to compile this dependency graph from source, the SPM flow consumes prebuilt XCFramework artifacts. In the implementation branch, the main packages exposed to the app are:

- `ReactNative`
- `ReactNativeDependencies`
- `hermes-engine`

These artifacts are downloaded during setup and cached locally, and the generated SPM package for the app links against them as binary targets. This keeps the app-facing package graph small and avoids moving the full complexity of the React Native native dependency tree into every application project.

At the implementation level, the branch also addresses one of the key technical challenges of using prebuilt artifacts with React Native: header compatibility. React Native still uses many CocoaPods-style include paths such as `#import <React/...>`, which SPM does not handle directly. The prototype solves this by combining:

- organized headers inside the XCFramework artifacts
- a generated Clang Virtual File System (VFS) overlay for the remaining non-standard include cases

This means the RFC does not need to argue only in theory that Hermes and React Native dependencies can work under SPM. The prototype already demonstrates a concrete model based on prebuilt XCFrameworks and local package generation.

### Build from source

Build React Native from source will be supported in the final version of the SPM package system, but the current POC does not support it.

### Third-party libraries

Third-party library support should follow the same general model as React Native core: libraries should be consumable through SPM, while still allowing an incremental migration period where CocoaPods and SPM coexist.

The simplest way forward is to add support for building using SPM by adding a Package.swift file to the library.

The current prototype already supports app-local native modules through `spmModules` in `react-native.config.js`. Each declared module is turned into a target in `autolinked/Package.swift`, with sources mirrored into the generated package layout so they can be compiled by SPM.

That mechanism is enough to prove that source-based native modules can be integrated into the SPM flow today.

For third-party libraries published to npm, the longer-term direction should be to add SPM metadata alongside the existing React Native autolinking metadata in `react-native.config.js`. That keeps React Native’s dependency discovery in one place and lets the CLI/autolinking pipeline decide how to integrate each library.

A library should be able to support one or both of these models:

- source-based SPM integration, where the library’s native sources are compiled as SPM targets
- prebuilt XCFramework distribution, where the library ships artifacts that are downloaded and linked during setup

A possible shape for library metadata is (properties inside the `spm` field are mutually exclusive):

```js
module.exports = {
  dependency: {
    platforms: {
      ios: {},
    },
  },
  spm: {
    xcframework: {
      name: 'ReactNativeMaps',
      url: 'https://example.com/react-native-maps-{version}-{flavor}.tar.gz',
    },{
    packageFile: {
      name: 'ReactNativeMaps',
      Path: 'ios/Package.swift'
    },
    source: {
      name: 'ReactNativeMaps',
      path: 'ios',
      publicHeadersPath: '.',
      exclude: ['*.podspec', 'Tests/**'],
    },
  },
};
```

The `spm` shape above is illustrative rather than final. It is included to show the kind of metadata React Native tooling would need in order to support both prebuilt and source-based integration.

In that model, React Native tooling could prefer prebuilt artifacts when available, and fall back to source compilation when needed. This is especially useful for libraries with heavy native build requirements, while still allowing smaller libraries to adopt SPM without setting up a prebuild pipeline immediately.

The current prototype does not yet implement full third-party npm library support through `spm.xcframework` or `spm.source`. What it does show is the important lower-level mechanism: generated SPM package targets, source mirroring for native code, and an autolinking pipeline that can evolve from app-local modules to ecosystem-wide library support.

During the transition period, libraries should be able to ship both:

- a `podspec` for CocoaPods users
- SPM metadata and, optionally, `Package.swift` or prebuilt XCFramework artifacts for SPM users

### Transition phases

The migration should happen in stages:

1. Experimental support: React Native ships opt-in SPM support for iOS.
2. Dual support: CocoaPods and SPM are both officially supported.
3. Default for new apps: once ecosystem coverage is good enough, SPM becomes the default for new iOS projects.
4. Deprecation: CocoaPods is deprecated in December 2026.
5. Removal of Cocoapods from the codebase: at some point after 12/2026. This depends on the rate of adoption by the community. We will keep it monitored as we did for the New Architecture rollout.

## Brownfield

Brownfield projects will also be able to use Package.swift files and integrate easily with existing Swift or Objective-C based apps, either as precompiled XCFrameworks or as source - both defined in a Package.swift file. There will be an option in the cli for generating separate Package.swift files for distributing the app as a brownfield target.

## Drawbacks

- React Native will need to support two iOS integration paths in parallel during the transition period, which increases maintenance cost and testing burden.
- Library maintainers may need to support both `podspec` and SPM metadata for a period of time.
- Some native modules may need restructuring before they can be expressed cleanly as SPM targets.
- Apps with highly customized Podfiles or brownfield iOS integration may need additional migration tooling.

## Alternatives

### Keep CocoaPods as the only supported path

This keeps the current model intact, but it preserves the Ruby/CocoaPods dependency and leaves React Native without a credible migration path ahead of the targeted December 2026 deprecation. This alternative requires that the community takes on maintaining Cocoapods - which doesn’t seem like a feasible way forward.

### Support SPM only for third-party dependencies, while keeping React Native core on CocoaPods

This reduces some integration pain, but it still leaves applications with two dependency systems for the same iOS target, which is harder to explain, document, and support.

### Support only prebuilt XCFrameworks and no source-based path

This simplifies some integration problems, but it would make adoption harder for smaller libraries and local development workflows. A source-based path should remain available even if prebuilt artifacts become the preferred distribution model.

## Adoption strategy

Adoption should be incremental rather than release-blocking.

- Ship SPM support behind an opt-in flow first.
- Add CLI and template support so new apps can be created or migrated with official tooling.
- Keep CocoaPods and SPM supported in parallel for several releases.
- Define a clear contract for third-party library authors.
- Encourage libraries to ship dual support during the transition period.
- Move SPM to the default for new apps only after ecosystem readiness is high enough.
- Communicate the migration plan to the ecosystem, including at App.js in May 2026, ahead of the targeted December 2026 deprecation.

This proposal is expected to be additive at first and only later become deprecating. Existing applications should not need to migrate immediately.

## How we teach this

The key teaching goal is to present SPM as the successor to the existing CocoaPods-based iOS setup, not as a completely different React Native model.

For app developers, the main concepts should be:

- React Native iOS can be integrated through either CocoaPods or Swift Package Manager during the transition period.
- The React Native CLI manages the SPM setup, Codegen, and autolinking flow.
- Apps consume React Native core, generated code, and autolinked modules as local Swift packages.
- There will be no difference in the UX for the developer - the implementation will be hidden.

For library authors, the main concepts should be:

- Continue supporting `podspec` during the transition period.
- Add SPM support through React Native library metadata and, where appropriate, `Package.swift` or prebuilt XCFramework artifacts.
- If possible, library developers should prefer prebuilt artifacts for heavy native dependency graphs, while keeping source-based integration available where practical.

Documentation should explain SPM as the default long-term iOS dependency model for React Native, with explicit migration guidance for existing apps and libraries.

## Unresolved questions

- What exact metadata shape should React Native standardize for third-party SPM support?
- When should SPM become the default for newly created iOS apps?
- What migration tooling is needed for brownfield apps and projects with heavily customized Podfiles?
- Which parts of the generated Xcode project flow can eventually move into more native SPM mechanisms as Apple tooling evolves?
- What minimum ecosystem readiness should be required before the CocoaPods deprecation is enforced?
