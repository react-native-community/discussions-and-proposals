---
title: Decoupling Metro
author:
- Rob Hogan (@robhogan)
date: 2023-04-17
---

# RFC0636: Decoupling Metro

## Summary

Put control of Metro versions in the hands of developers by decoupling it from the `react-native-community/cli` and `facebook/react-native` release schedules, and making it a project-level dependency. Allow features and fixes to reach React Native users faster, and shorten the feedback loop.

## Basic example

If the proposal involves a new or changed API, include a basic code example. Omit this section if it's not applicable.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Please focus on explaining the motivation so that if this RFC is not accepted, the motivation could be used to develop alternative solutions. In other words, enumerate the constraints you are trying to solve without coupling them too closely to the solution you have in mind.

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

### Stage 3 - Add `metro` to Template dependencies and make it a peer dependency of RN and RNCLI.

## Drawbacks

### Template churn
Any change to the Template increases the friction of upgrading React Native, which is already a commonly-reported pain point.

### Fewer batteries included
React Native effectively comes with an integrated bundler

## Alternatives

### High level
Rather than `metro` being a project dependency, it could instead be a `react-native` dependency.

### Details
There are various alternatives in the details of the design - for example, either or both of `@react-native/babel-preset` and `@react-native/metro-babel-transformer` may be 

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

## How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers?

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?
