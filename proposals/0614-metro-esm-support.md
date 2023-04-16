---
title: Improved support for ECMAScript modules
author:
- Moti Zilberman
date: 2023-04-17
---

# RFC0614: Improved support for ECMAScript modules

## Summary

Enabling built-in support for ECMAScript modules in Metro for all React Native projects, fixing limitations in the current setup.

## Motivation

Today, [`metro-react-native-babel-preset`](https://github.com/facebook/metro/tree/main/packages/metro-react-native-babel-preset) includes the standard [Babel ESM-->CommonJS plugin](https://reactnative.dev/docs/ram-bundles-inline-requires#inline-requires) which compiles `import`, `export` etc into `require()`.

There are a few problems in this model:

* **Problem 1:** [Inline requires](https://reactnative.dev/docs/ram-bundles-inline-requires#inline-requires) do not work reliably for `import` declarations ([metro#909](https://github.com/facebook/metro/issues/909)). There is a separate, undocumented preset option, `lazyImportExportTransform`, that controls `import` inlining via the Babel plugin's [`lazy`](https://babeljs.io/docs/babel-plugin-transform-modules-commonjs#lazy) option.
* **Problem 2:** `Platform` inlining doesn't work for the common case of `import {Platform} from 'react-native'` ([metro#800](https://github.com/facebook/metro/pull/800#issuecomment-1092434015)).
* **Problem 3:** [Lazy bundling](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0605-lazy-bundling.md) does not work, since Babel transforms away `import()` calls before Metro can process them.

These problems lead to larger bundle sizes, worse runtime performance, and slow development speed.

The unstable `experimentalImportSupport: true` flag in Metro enables built-in ESM support similar to Babel's (notably, with the same `__esModule: true` interop) that solves the above problems. This has been in use at Meta for the last few years. This proposal aims to make this the default behaviour in Metro in a way that is least disruptive.

Any change to the semantics and ordering of module initialisation is breaking, so this proposal also includes an opt-out mechanism and warnings when the build is misconfigured.

## Detailed design

### Core changes

The `experimentalImportSupport` flag will be removed and have no effect on Metro's output.

The `inlineRequires: true` flag will remain in the default React Native [Metro config](https://github.com/facebook/react-native/blob/f5c060618584ae736987b48557d69c501547f9d4/packages/metro-config/index.js#L73).

`@babel/plugin-transform-modules-commonjs` will be removed from `metro-react-native-babel-preset`, and with it the `lazyImportExportTransform` and `disableImportExportTransform` preset options, which will no longer have an effect on the preset's output.

> _Alternative_: Rename `disableImportExportTransform` to `enableImportExportTransform` with a default of `false`.

Metro will always apply its built-in ESM transform after any user-specified Babel plugins. If the user's Babel config includes a different ESM transform, Metro's built-in one will have no effect.

In the course of implementing this, we will expand the test coverage for Metro's built-in ESM transform to ensure parity and compatibility with Babel's version.

As a result of these changes, we will no longer need to dynamically set `disableImportExportTransform` from `metro-react-native-babel-transformer` when enabling the built-in ESM transform, which is a brittle mechanism that users can easily override accidentally today.

### Opt-out and warnings

In case the new `import` inlining causes issues, users can revert to the old behaviour by manually adding `@babel/plugin-transform-modules-commonjs` to their Babel configs. Note that it may not be the *exact* old behaviour (because Metro currently has [custom defaults](https://github.com/facebook/metro/blob/main/packages/metro-react-native-babel-preset/src/configs/main.js#L66-L68) for the `lazy` Babel option), but it will be *possible* to configure the plugin to closely model the old behaviour as needed.

For most users, a better option than reverting will be to blocklist specific modules from the inlining behaviour, or better yet, fix the reliance on implicit module side effects. We will provide easy-to-follow guidance and include in the release announcement.

If any of the removed config flags (`experimentalImportSupport`, `disableImportExportTransform` and `lazyImportExportTransform`) are detected, Metro will show warnings in the terminal with relevant instructions on how to migrate the Metro config correctly. The flags will not have an effect on Metro's output.

If Metro detects that a user-specified Babel plugin has removed all `import` and `export` AST nodes from a module, it will show a warning in the terminal recommending that the user remove the plugin and use the built-in support for ESM.

The warnings described above will only be shown once per instance of Metro, to avoid spamming the terminal.


## Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React Native
- integration of this feature with other existing and planned features
- cost of migrating existing React Native applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? Why did you select your approach?

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

TODO: Jest + preset changes

## How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers?

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?
