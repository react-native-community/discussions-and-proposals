---
title: Decoupling Metro
author:
- Rob Hogan (@robhogan)
date: 2023-04-17
---

# RFC0636: Decoupling Metro

## Summary

Put control of Metro versions in the hands of developers by decoupling it from the `react-native-community/cli` and `facebook/react-native` release schedules, and making it a project-level dependency. Allow features and fixes to reach React Native users faster, and shorten the feedback loop.

## High-level

### Before
<img width="470" alt="image" src="https://user-images.githubusercontent.com/2590098/232753387-5ac49576-be87-4ea4-bac8-906cb3322b7e.png">

### After

<img width="633" alt="image" src="https://user-images.githubusercontent.com/2590098/232753653-e1e31695-c9ef-45f7-ad5f-a224c4efe6ac.png">

## Motivation

TODO

## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

### Terminology
 - **Metro Packages**: NPM packages published from the [`facebook/metro` monorepo](https://github.com/facebook/metro), that is `metro`, packages with a `metro-` prefix, and `ob1`.
 - **RNCLI**: 
 - **Template**: The template used for projects initialised with `react-native-community/cli`, distributed as part of `react-native`.
 - **RN Monorepo**: The monorepo at [`facebook/react-native`](https://github.com/facebook/react-native), the source of `react-native` and `@react-native/*` NPM packages.

### Stage 1 - Move RN-coupled Metro packages to the RN Monorepo.

***Goal: Relocate Metro Packages that are tied to React Native versioning to allow for loose RN-Metro coupling***

**`metro-react-native-babel-preset` -> `@react-native/babel-preset`**

*Why does it belong in the RN Monorepo?*

 - It's a conformant [Babel preset](https://babeljs.io/docs/presets#creating-a-preset) not inherently coupled to Metro in any way (hence proposed name does not include "metro").
 - Used by Jest in the RN Monorepo to test React Native code, unrelated to Metro.
 - Designed so that the output is minimally downlevelled to support all React Native target runtimes / compilers. When the preset is modified it's either to reflect changes in those capabilities or to accommodate new features for RN developers.

 *Decoupling from Metro*

 This package has no dependencies on Metro Packages, and is only depended upon by `metro-react-native-babel-transformer` and the Template, so it can be moved alongside the transformer.

**`metro-react-native-babel-transformer` -> `@react-native/metro-babel-transformer`**

*Why does it belong in the RN Monorepo?*

 - Uses `metro-react-native-babel-preset`
 - Must apply `react-refresh/babel` using the same version of `react-refresh` as is injected into the React Native runtime within `react-native` (details below).
 - Implements the Metro Babel transformer "interface" with customisations specific to React Native - these may change with React Native's capabilities.

*Decoupling from Metro*

`metro-react-native-babel-transformer` has runtime and type dependencies on `metro-source-map`, and type dependencies on `metro-babel-transformer`. These may be eliminated by inversion or injection to allow the package to move to the RN monorepo.

**Note on Fast Refresh - dependencies on `react-refresh`**

Currently, each of `metro-runtime`, `metro-react-native-babel-transformer` and `react-native` have dependencies on `react-refresh` that *must align* - this has made upgrading it problematic in the past. 

The concrete dependencies are in the transformer (for `react-refresh/babel`) and in [`react-native/Libraries/Core`](https://github.com/facebook/react-native/blob/72d2880999a64957156f80ca61254af991252f51/packages/react-native/Libraries/Core/setUpReactRefresh.js#L22) (to inject `react-refresh/runtime`). `metro-runtime`'s two dependencies are:
 1. Indirect, via calls to the `__ReactRefresh` global API [defined in RN](https://github.com/facebook/react-native/blob/72d2880999a64957156f80ca61254af991252f51/packages/react-native/Libraries/Core/setUpReactRefresh.js#L24-L50) (or other Metro clients) and,
 2. Explicit, to re-export `metro-runtime/refresh-runtime` for RN's benefit.

(1) can remain as-is since the contract is between Metro and client runtime, which abstracts and extends `react-refresh`, rather than with `react-refresh` directly. (2) is not yet in used and can be removed, though it will do no harm if its version diverges.

### Stage 2 - Eliminate direct dependencies on `metro-*` subpackages.
***Goal: Minimise impact to user template and allow for a single "source of truth" for Metro's version in a given project.***

Example: `@react-native-community/cli-plugin-metro` depends on `metro`, `metro-runtime`, and `metro-config` - these are listed separately in its `package.json` and should always align. If we were to move these to `peerDependencies` (stage 3) they would have to all be individually listed in the user project's `package.json`, which creates noise and the potential for mismatch.

Design: Re-export necessary types and APIs from `metro` and use it as an umbrella package, at least for the needs of RNCLI and RN.

### Stage 3 - Add `metro` to Template dependencies and make it a peer dependency of RN and RNCLI.
***Goal: User control of Metro version, within peer constraints set by React Native/Metro***

Design: Add `metro` to Template `package.json` under `dependencies`. Within RN and RNCLI, move `metro` from `dependencies` or `devDependencies` into `peerDependencies` with a reasonably permissive version constraint.

## Drawbacks

- **Template churn**: Any change to the Template increases the friction of upgrading React Native, which is already a commonly-reported pain point.
- **Fewer batteries included**: React Native effectively has an "integrated" bundler, this would loosen the integration and perhaps perceived cohesion. However, in practice the barrier to installing Metro as a peer of RN is low, and installing a bundler alongside a UI framework is closer to the industry norm.

## Alternatives

### High level
  - **`metro` as a `react-native` dependency rather than a Template dependency**: This addresses the Template churn and integration drawbacks above, but falls short of the goal of getting Metro releases to users as soon as they're available.

## Adoption strategy

Stage 1 is not a breaking change - though we would cease publishing new versions of `metro-react-native-babel-preset` (etc.), existing users would be able to continue to use the published versions. There would be no immediate need to update their projects.

Stage 2 is an internal change with no impact on users.

Stage 3 will require that users update their projects to add `metro` to their `dependencies` - we could only introduce this with a major React Native release. However, package managers that don't auto-install peers (pnpm, yarn, npm<7) should clearly alert users to a missing peer dependency, and we can additionally mitigate by checking for the presence of Metro on RNCLI (or Expo CLI) launch, failing gracefully with instructions.

## How we teach this

TODO

## Unresolved questions

 - ***Separate packages?*** Either or both of `@react-native/babel-preset` and `@react-native/metro-babel-transformer` could, instead of being new packages, simply be directories within `react-native`. The tradeoffs here are mostly "internal" to contributors and the release process.

 - ***Re-export babel-preset?*** If `@react-native/babel-preset` *is* a separate package, we could either make it a Template dependency - a direct swap for the current `metro-react-native-babel-preset` dependency, or re-export it from `react-native/babel-preset`.

 - ***`@react-native/metro-config`*** Similarly, it's unclear whether this needs to be a Template dependency under the current setup or whether we could re-export the necessary APIs from `metro` and `react-native`.