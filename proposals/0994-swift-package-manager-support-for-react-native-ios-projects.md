---
title: Swift Package Manager Support for React Native iOS Projects
author:
  - Christian Falch (@chrfalch)
date: 2026-04-09
---

# RFC0994: Swift Package Manager Support for React Native iOS Projects

## Summary

This RFC proposes adding first-class support for Swift Package Manager (SPM) for React Native on Apple platforms, with the goal of replacing CocoaPods over a staged transition period (see the CocoaPods blog post, [CocoaPods Trunk Read-only Plan](https://blog.cocoapods.org/CocoaPods-Specs-Repo/)).

With CocoaPods deprecation targeted for December 2026, React Native needs a supported SPM-based integration story for React Native core, Hermes, Codegen, autolinking, and third-party native libraries. Cocoapods will be read-only and prevent creation of new libraries.

## Motivation

React Native’s iOS integration is currently tightly coupled to CocoaPods. This creates several issues:

- React Native apps require a Ruby/CocoaPods toolchain in addition to the standard Xcode toolchain.
- Some Apple ecosystem dependencies are now distributed primarily through Swift Package Manager. They can still be used in the current setup, but with some limitations.
- React Native library authors must maintain CocoaPods integration even when SPM is the preferred native packaging format.
- The current setup diverges from the default dependency workflow used by many Apple developers.

If React Native intends to deprecate CocoaPods in December 2026, it needs a migration path that is practical for app developers, library maintainers, and the CLI/tooling ecosystem. The motivation for this RFC is therefore not just to add another packaging option, but to define a credible path from today’s CocoaPods-based workflow to a supported SPM-based workflow.

## Basic example

For a new project, SPM is set up with:

```bash
npx react-native spm  # invokes node node_modules/react-native/scripts/setup-apple-spm.js
                      # with no action — auto-resolves to the right verb (see below)
```

`spm` is registered as a React Native CLI command, so no separate installation
step is required.

The command exposes four developer-facing verbs; with no action it auto-resolves which one to run:

- `add` — runs React Native Codegen, generates
  `build/generated/autolinking/Package.swift`, downloads the prebuilt XCFramework
  artifacts, generates the local SPM packages, and **injects** the SPM package
  references into the app's existing Xcode project, in place. The injection is
  additive and idempotent, and is recorded in a `.spm-injected.json` marker.
  Default on a first run.
- `update` — re-runs the pipeline and refreshes the existing injection. Default
  once a project has been set up.
- `deinit` — the exact inverse of `add`: surgically removes only what `add`
  injected (using the marker) and drops it. Git-recoverable; no prompt.
- `scaffold` — generates a `Package.swift` for community libraries that ship only
  a podspec.

Three further verbs exist for tooling rather than day-to-day use: `sync` (the
lightweight resync the Xcode hooks call), `codegen`, and `download`.

For an app that still uses CocoaPods, `add --deintegrate` first runs
`pod deintegrate` and strips React Native from the Podfile, then injects SPM —
converting the project in one step. The zero-arg `npx react-native spm` detects a
freshly-scaffolded CocoaPods project and does this automatically; on an
established app it stops and points the developer at `--deintegrate`. To remove
SPM again, `npx react-native spm deinit` (then `pod install` to return to
CocoaPods). Because the integration is injected in place rather than generated
from scratch, the app keeps its own `.xcodeproj` (signing, capabilities, build
phases) — no project is renamed or replaced.

## Detailed design

React Native should support two iOS integration flows during the migration period:

- CocoaPods, as the existing integration path.
- Swift Package Manager, as the new first-class integration path.

### React Native core packaging

The implementation already provides a working SPM packaging model.

React Native core reaches the app through prebuilt XCFrameworks, but — importantly — **the runtime binaries are not members of the Swift package graph.** SwiftPM binary targets cannot vary by build configuration, and React Native ships *flavored* binaries: the debug `React.framework`, `hermes-engine`, and `ReactNativeDependencies` carry the dev experience (dev menu, assertions), while the release builds strip it. A Debug build must embed the debug binaries and a Release build the release ones, which a `binaryTarget` cannot express.

The design therefore splits the two concerns:

- **Headers travel through SwiftPM.** A single `ReactNative` package vends three headers-only products — `ReactHeaders`, `ReactNativeHeaders`, and `ReactNativeDependenciesHeaders` — so app and library targets compile against React Native's API through ordinary product dependencies.
- **Runtime binaries travel outside SwiftPM.** Setup downloads and validates both flavors into immutable app-local slots, then injects SDK/architecture-qualified Xcode build settings that link the exact selected binaries, plus one build phase that copies and signs the selected frameworks into the app. Configurations named `debug`/`development` select Debug; all others select Release.

Selection uses only generated build settings and standard tooling: a build does not run Node, mutate symlinks, regenerate the package graph, or require a second pass.

The prototype uses a single script, `setup-apple-spm.js`, to prepare an app for this layout. That script:
runs Codegen,
generates SPM autolinking,
downloads the required artifacts,
generates the package manifests, and
injects the SPM package references into the app's existing Xcode project, in place.

From the app’s point of view, the integration stays relatively simple: the app's Xcode project references three locally generated packages — React Native core, generated code, and autolinked native modules — through `XCLocalSwiftPackageReference` entries. The injection also adds auto-sync hooks that refresh autolinking when dependencies change: a pre-action on the app's shared scheme, backed by a build phase as a safety net for builds that bypass the scheme. No app-level `Package.swift` is generated or required; the sub-package paths under `build/` are stable, so adding or removing community deps never requires re-injecting. The edit is purely additive and recorded in a `.spm-injected.json` marker, so it can be reversed exactly (`deinit`) without touching the rest of the project.

This RFC does not propose a new packaging model from scratch. Instead, it proposes standardizing and evolving the direction the implementation already demonstrates. The tooling, commands, header model, and extension points described below are documented under `packages/react-native/scripts/spm/__docs__/` (start with `spm-scripts.md`).

### Autolinking

The current prototype keeps the React Native autolinking model, but changes the output from CocoaPods configuration to Swift Package Manager configuration.

Instead of generating Podfile integration, the SPM flow generates a local `build/generated/autolinking/Package.swift` that collects the native React Native dependencies discovered in the app. The generated Xcode project references this package directly via `XCLocalSwiftPackageReference`, alongside the generated Codegen package and the React Native core package.

At a high level, the flow is:

1. React Native Codegen generates the native metadata and codegen artifacts for the app.
2. The SPM autolinking step reads the generated autolinking metadata and the app’s `react-native.config.js`.
3. It produces `build/generated/autolinking/Package.swift`, which contains local package targets for the app’s native dependencies.
4. The Xcode project references that package, so native modules are available through normal SPM package resolution.

This keeps the developer-facing model close to today’s React Native workflow: dependencies are still discovered from the JavaScript project, but the integration backend is SPM instead of CocoaPods.

### Local modules

The prototype also supports additional local modules through `spm.modules` in `react-native.config.js`. This makes it possible to include native code that is not otherwise discovered through standard autolinking.

For example:

```js
module.exports = {
  spm: {
    modules: [
      {
        name: "MyNativeModule",
        path: "ios/MyNativeModule",
        exclude: ["*.podspec"],
        publicHeadersPath: ".",
      },
    ],
  },
};
```

For a new project, the initial setup is performed with:

```bash
npx react-native spm
```

This first-time setup generates the SPM sub-packages and injects them into the app's existing Xcode project in place. On a CocoaPods project, `npx react-native spm add --deintegrate` removes CocoaPods first (and the zero-arg command does this automatically when it detects a freshly-scaffolded CocoaPods app), so no project is renamed and `npm run ios` resolves to the same `.xcodeproj`.

After initialization, autolinking is normally kept up to date automatically. The auto-sync hooks detect dependency changes and regenerate the autolinked package before compilation when needed.

When a manual refresh is needed, for example after changing native dependencies or local module configuration, developers can re-run:

```bash
npx react-native spm   # resolves to `update` once the project is set up
```

To remove the SPM integration again, `npx react-native spm deinit` reverses exactly what `add` injected.

### Codegen

The current prototype keeps React Native Codegen as part of the normal iOS setup flow, but packages its output for Swift Package Manager instead of CocoaPods.

In the implementation branch, `setup-apple-spm.js` starts by running the existing React Native Codegen pipeline and writing the generated output to `build/generated/ios/`. After that, it installs an SPM-specific `Package.swift` into that generated directory so the output can be consumed as a local Swift package.

From the app’s point of view, the generated code is exposed through a separate package, `React-GeneratedCode`, with products such as `ReactCodegen` and `ReactAppDependencyProvider`. The Xcode project references that generated package alongside the React Native core package and the autolinked package.

This keeps Codegen in roughly the same place in the developer workflow as today: it still runs before native compilation, but its output is expressed as SPM package targets rather than Pod targets.

We’ll also port the current detection of changes in the Spec files to the new SPM packaging system.

### Hermes and native dependencies

The current prototype already includes a concrete packaging model for Hermes and React Native’s native third-party dependencies.

Instead of asking each app to compile this dependency graph from source, the SPM flow consumes prebuilt XCFramework artifacts, downloaded during setup and cached locally. Three runtime artifacts are staged per flavor — `React.xcframework`, `ReactNativeDependencies.xcframework`, and `hermes-engine.xcframework` — and linked and embedded through Xcode build settings rather than as SwiftPM binary targets, for the configuration-selection reason described above. This keeps the app-facing package graph small and avoids moving the full complexity of the React Native native dependency tree into every application project.

The implementation also addresses one of the key technical challenges of using prebuilt artifacts with React Native: header compatibility. React Native uses framework-style includes such as `#import <React/...>`, many lowercase namespace includes such as `#import <react/...>`, `<yoga/...>`, and `<jsi/...>`, and the third-party dependency namespaces (`folly/`, `glog/`, `boost/`, …) — none of which SPM serves directly from a plain framework binary. Three headers-only products solve this, each owning a distinct set of namespaces:

- `ReactHeaders`, a Clang target over a canonical header tree staged at setup time, serving the Objective-C `#import <React/...>` and Swift `import React` surface through a plain `module React` module map. The headers are staged only after Debug and Release are proven to expose identical public headers, so one copy is valid for both flavors
- `ReactNativeHeaders.xcframework`, a headers-only artifact whose module map makes React Native's own lowercase namespaces (`react/`, `yoga/`, `jsi/`, `ReactCommon/`, …) modular — kept separate from `ReactHeaders` so that lowercase `react` and Objective-C `React` never collide
- `ReactNativeDependenciesHeaders.xcframework`, a headers-only sidecar for the dependency namespaces (`folly/`, `glog/`, `boost/`, …). The `ReactNativeDependencies` binary is a framework and cannot expose those headers to SwiftPM, so the headers ship as their own library-type artifact

Targets take these as ordinary product dependencies, so every include above resolves with **no header search paths at all** — no `-I`/`-F` flags and no `unsafeFlags` in any generated manifest — and with no per-app configuration.

(An earlier iteration used a generated Clang Virtual File System overlay for the non-standard includes; that approach was replaced by the headers-only products above, which need no per-app overlay generation.)

This means the RFC does not need to argue only in theory that Hermes and React Native dependencies can work under SPM. The implementation already demonstrates a concrete model based on prebuilt XCFrameworks and local package generation.

### Build from source

Build React Native from source will be supported in the final version of the SPM package system, but the current POC does not support it.

### Third-party libraries

Third-party library support should follow the same general model as React Native core: libraries should be consumable through SPM, while still allowing an incremental migration period where CocoaPods and SPM coexist.

The simplest way forward is to add support for building using SPM by adding a Package.swift file to the library.

The current prototype already supports app-local native modules through `spm.modules` in `react-native.config.js`. Each declared module is turned into a target in `build/generated/autolinking/Package.swift`, with sources mirrored into the generated package layout so they can be compiled by SPM.

A library that needs *another* native library declares that with `spm.dependencies` in its own `react-native.config.js`, a list of npm names that the autolinker follows recursively. SwiftPM has no equivalent of a podspec's `s.dependency`, so the relationship has to be stated explicitly.

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
    // Exactly one of the three below.

    // 1. Prebuilt XCFramework, downloaded at setup time.
    xcframework: {
      name: 'ReactNativeMaps',
      url: 'https://example.com/react-native-maps-{version}-{flavor}.tar.gz',
    },

    // 2. A Package.swift the library ships itself.
    packageFile: {
      name: 'ReactNativeMaps',
      path: 'ios/Package.swift',
    },

    // 3. Sources compiled as an SPM target.
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

#### Tooling for library authors

React Native already has a prebuild pipeline (`scripts/ios-prebuild/`) that produces packaged XCFrameworks. Rather than asking every library author to reinvent it, that tooling could be exposed as a reusable CLI, so that shipping a prebuilt artifact is a CI step rather than a project:

```bash
npx react-native-prebuild \
  --podspec ios/MyLibrary.podspec \
  --react-native-version 0.80.0 \
  --platforms ios,ios-simulator \
  --flavor release \
  --output dist/
```

It would download the React Native artifacts for the target version, read the library's podspec for sources, headers, and flags, generate a temporary manifest, build each platform slice, compose the XCFramework with organized headers and module map, and package it with its dSYMs. Since unsigned artifacts trigger Gatekeeper warnings for consumers, code signing with the author's own identity belongs in the tool rather than in each author's CI.

This is complementary to, not required by, the rest of the proposal: a library can adopt SPM by shipping a `Package.swift` and compiling from source, and reach for prebuilt artifacts only when its native build is heavy enough to justify them.

### Framework plugins (autolinking hooks)

Some ecosystems — Expo is the first — layer their own module system on top of React Native: they discover native modules dynamically, generate a module registry, and ship mixed Swift/Objective-C/C++ modules such as `ExpoModulesCore`. A static `spm.modules` list cannot express this, and a one-shot post-process of the generated `Package.swift` would be overwritten the next time the auto-sync hooks re-run autolinking.

The prototype therefore exposes a generic **autolinking plugin hook**. Any dependency can register a plugin from its own `react-native.config.js` (a `spm.autolinkingPlugin` entry), and React Native invokes it wherever autolinking runs — during `add`/`update` and during the build-time sync — so a framework's contribution is regenerated on every build and never goes stale. The plugin receives the app/JS-root context and the discovered autolinking metadata, and returns SwiftPM package references, product dependencies, generated-source files, precompiled Debug/Release framework pairs, and the extra inputs the build-time sync must watch, all of which React Native merges into the autolinked package graph. A further field for build-time script phases — SwiftPM has no equivalent of CocoaPods' `script_phase` — is in review. Discovery is transitive — installing the framework is enough, mirroring how CocoaPods pulls in `use_expo_modules!` — with an app-level opt-out for excluding a plugin.

This keeps React Native framework-agnostic (no framework-specific code in core) while giving frameworks the same extension seam CocoaPods provided through the Podfile, `use_expo_modules!`, and its post-install hooks. Expo is the first consumer and is validating the contract, which ships as an unstable preview until proven. The discovery mechanism and the plugin contract are documented in `packages/react-native/scripts/spm/__docs__/spm-autolinking-plugins.md`.

### Transition phases

The migration should happen in stages:

1. Experimental support: React Native ships opt-in SPM support for iOS.
2. Dual support: CocoaPods and SPM are both officially supported.
3. Default for new apps: once ecosystem coverage is good enough, SPM becomes the default for new iOS projects.
4. Deprecation: CocoaPods is deprecated in December 2026.
5. Removal of Cocoapods from the codebase: at some point after 12/2026. This depends on the rate of adoption by the community. We will keep it monitored as we did for the New Architecture rollout.

## Brownfield

Because `add` injects into the app's existing Xcode project in place (rather than generating one), an app that embeds React Native can adopt SPM today by pointing `add` at the right project and target (`--xcodeproj` / `--product-name`); the current constraint and coexistence rules are described in the tool docs (`spm-scripts.md`, "Brownfield apps"). Building on that, brownfield projects will also be able to use Package.swift files and integrate easily with existing Swift or Objective-C based apps, either as precompiled XCFrameworks or as source — both defined in a Package.swift file. There will be an option in the cli for generating separate Package.swift files for distributing the app as a brownfield target.

## Drawbacks

- React Native will need to support two iOS integration paths in parallel during the transition period, which increases maintenance cost and testing burden.
- Library maintainers may need to support both `podspec` and SPM metadata for a period of time.
- Some native modules may need restructuring before they can be expressed cleanly as SPM targets.
- Apps with highly customized Podfiles or brownfield iOS integration may need additional migration tooling.
- Because the generated packages live in a gitignored directory, a clean checkout has nothing to resolve until they are regenerated. A one-time setup run — the analog of `pod install` — is therefore still required on fresh clones and in CI. A hosted manifest with remote binary targets would remove this step; see unresolved questions.
- Both Debug and Release flavors of the runtime artifacts are downloaded and staged, which doubles the artifact footprint on disk and on first fetch.

## Alternatives

### Keep CocoaPods as the only supported path

This keeps the current model intact, but it preserves the Ruby/CocoaPods dependency and leaves React Native without a credible migration path ahead of the targeted December 2026 deprecation. This alternative requires that the community takes on maintaining Cocoapods - which doesn’t seem like a feasible way forward.

### Support SPM only for third-party dependencies, while keeping React Native core on CocoaPods

This reduces some integration pain, but it still leaves applications with two dependency systems for the same iOS target, which is harder to explain, document, and support.

### Support only prebuilt XCFrameworks and no source-based path

This simplifies some integration problems, but it would make adoption harder for smaller libraries and local development workflows. A source-based path should remain available even if prebuilt artifacts become the preferred distribution model.

### Use SwiftPM plugins instead of injected Xcode build phases

SwiftPM's own plugin types were evaluated as a way to avoid touching the app's Xcode project at all. They do not fit React Native's build steps today:

- **Sandboxing.** Plugins run sandboxed. A build tool plugin may only write to its designated work directory, not the source tree, and cannot freely run `node` or read `node_modules` — all of which the autolinking and codegen steps require.
- **No Xcode build settings.** Plugins do not receive `CONFIGURATION`, `PLATFORM_NAME`, or `BUILT_PRODUCTS_DIR`. The JS bundling step relies on these to decide debug-vs-release behavior and output paths, and flavored framework selection is expressed entirely in build settings.
- **No post-build stage.** JS bundling and Hermes bytecode compilation place artifacts into the built `.app`. SwiftPM has no post-build plugin capability, by design, for build reproducibility.

A hybrid (some plugins, some build phases) would be harder to reason about than the current uniform approach. SwiftPM plugins shine for pure source generation, which is not the shape of this problem. This is worth revisiting if Apple exposes build configuration and broader I/O to plugins.

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
- **How should a library's React Native version compatibility be enforced?** A prebuilt library XCFramework is built against a particular React Native version. Should setup enforce strict matching, accept a semver range, or warn on mismatch? Embedding the React Native version, library version, flavor, and minimum iOS version in the artifact makes the check possible; the policy is the open part.
- **How should debug symbols be distributed and consumed?** The prebuild pipeline produces dSYM bundles, but not a distribution story. Should they download alongside the XCFrameworks or on demand? How should they reach crash reporters (Sentry, Crashlytics) that need dSYM UUIDs to symbolicate? Should setup place them where Xcode's archive workflow finds them automatically?
- **How should library authors validate SPM compatibility in CI?** A validation mode that compiles a library's sources as an SPM target without producing a release artifact would let maintainers gate pull requests on SPM support without running a full prebuild.
- **Where should library XCFrameworks be hosted?** Maven (consistent with React Native core), GitHub Releases (simplest for authors), or a dedicated registry (best discovery and compatibility metadata) trade off differently on reliability and maintenance burden.
- **How should a hosted manifest be distributed?** Local path-based packages require a setup run before a clean checkout can resolve. A tagged manifest repository with `binaryTarget(url:checksum:)` would let SwiftPM fetch artifacts during ordinary resolution and remove that step; the implementation already supports a remote mode behind an opt-in, but the hosting location, tagging policy for nightlies, and flavor handling are unresolved.
