---
title: Move core-cli-utils to private/ as a reference implementation
author:
  - Alex Hunt
date: 2026-05-21
---

# RFC1002: Move `core-cli-utils` to `private/` as a reference implementation

## Summary

Move `@react-native/core-cli-utils` from `packages/` to `private/`, stop publishing it to npm, and reframe it as a reference implementation of React Native Framework CLI tooling rather than a public API.

## Motivation

`@react-native/core-cli-utils` was created as a library for framework authors to build their CLI tooling on top of. In practice, it has no known external consumers and is only used internally by two packages:

- **`private/helloworld/`** — the primary and most complete consumer, using the `android`, `apple`, and `app` (Metro bundler) modules.
- **`packages/rn-tester/`** — uses only `apple.bootstrap()` for iOS CocoaPods setup.

The package has **zero declared dependencies** (relying on monorepo workspace hoisting), **no tests**, and its README describes a hypothetical "fancy-framework" use case that doesn't reflect any real adoption. Publishing it to npm creates a maintenance surface that isn't justified:

- Any future framework adopter would likely fork the code and evolve it independently, rather than depend on a versioned API.
- Versioning, changelogs, and compatibility cost effort with no external users to benefit.

The code itself is valuable — it encodes working knowledge of how to drive Gradle, xcodebuild, CocoaPods, and Metro from a CLI. Moving it to `private/` preserves that value as a reference implementation while removing the pretense that it's a stable public API.

## Detailed design

### 1. Move the package

Move `packages/core-cli-utils/` to `private/core-cli-utils/`.

Remove it from the npm publish pipeline in `scripts/build/config.js`. Remove the `publishConfig` field from its `package.json`. Remove its entry from `xplat/js/public/package.json` workspaces (or update the path).

### 2. Update consumers

**`private/helloworld/`**: Update the import paths and `devDependencies` entry to point to the new location. Since both packages are now under `private/`, this can use a workspace-relative path.

**`packages/rn-tester/`**: rn-tester uses `apple.bootstrap()` and the `Task` type. Two options:

- **(a) Keep the dependency** — rn-tester already depends on several `private/` packages and internal-only code. A cross-boundary dependency on `private/core-cli-utils` is consistent with its role as an internal test app.
- **(b) Inline the bootstrap logic** — rn-tester's usage is minimal (one call site). The `apple.bootstrap()` tasks could be inlined into `rn-tester/cli.flow.js` directly, removing the dependency entirely.

Option (a) is recommended to avoid code duplication and keep the reference implementation as the single source of truth for CLI task definitions.

### 3. Update documentation

Update `core-cli-utils/README.md` to describe the package as a reference implementation of React Native CLI tooling. Remove npm badge and "Contributing" section that references publishing. Add a note that framework authors can use this code as a starting point but should not depend on it as a versioned API.

### 4. Clean up

Remove the package from:

- The monorepo build config (`scripts/build/config.js`)
- The desktop import config (`xplat/js/tools/desktop/config/import.js`)
- The `patchCoreCLIUtilsPackageJSON()` helpers in `rn-tester/scripts/monorepo.js` and `helloworld/scripts/monorepo.js` (these exist to patch entry points for monorepo vs. published usage — no longer needed if the package is never published)

## Drawbacks

- **Discoverability for framework authors**: If a framework author wanted to build CLI tooling on top of React Native's primitives, a published npm package is easier to discover than code under `private/`. However, no framework author has done this to date, and the Community CLI (`@react-native-community/cli`) implements its own build/run commands independently.

- **Future reversal**: If demand for a public CLI utilities package emerges, moving it back to `packages/` and re-publishing requires restoring the build config and versioning. This is straightforward but not free.

## Alternatives

1. **Keep as-is**: Continue publishing to npm with no external users. Low cost, but adds noise to the release process and signals a public API that isn't maintained as one.
2. **Delete entirely, inline into helloworld**: Move the source files directly into `private/helloworld/lib/` and duplicate the small subset rn-tester needs. Eliminates the package boundary but loses the reference implementation framing.
3. **Invest in making it a real public API**: Add tests, declare dependencies properly, document the API, and promote adoption by framework authors. This is the original vision but requires significant investment with uncertain demand.

## Adoption strategy

This is not a breaking change for any external app consumers. The package has no known users outside the React Native monorepo. The npm package can be deprecated with a notice pointing to the source code under `private/` for anyone who wants to use it as a reference.
