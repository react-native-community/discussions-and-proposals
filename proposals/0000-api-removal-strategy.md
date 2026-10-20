---
title: API Removal Strategy in React Native
author:
  - Satyajit Sahoo
date: 2026-10-22
---

# RFC0000: API Removal Strategy in React Native

## Summary

This document intends to discuss strategy for removing components, modules etc. in React Native core.

## Motivation

A while ago `InteractionManager` was removed from React Native core, and [the getter now throws an error when accessing it](https://github.com/react/react-native/pull/57026/changes). This approach makes it difficult for library developers to gracefully handle this change. As the behavior of getters differs according to how React Native is imported, the bundler etc.

Let's consider the following scenario when trying to detect if `InteractionManager` is available:

```jsx
import * as ReactNative from 'react-native';

let InteractionManager;

try {
  InteractionManager =
    'InteractionManager' in ReactNative
      ? ReactNative.InteractionManager
      : undefined;
} catch (e) {
  // handle gracefully
}
```

The `import * as ReactNative` statement will throw an error as the getters are evaluated immediately making them eager in the following configuration:

- Metro `experimentalImportSupport: true`
- Babel `disableImportExportTransform: true`

Many apps are using this combination such as [Bluesky](https://github.com/bluesky-social/social-app/pull/1772). It's especially worse because `import * as ReactNative` itself throws, even if there is no mention of `InteractionManager`, potentially breaking apps that don't even use it.

Here is [another scenario](https://github.com/software-mansion/react-native-gesture-handler/pull/4493) where the default setup triggers warnings due to eager evaluation of imports.

An alternative approach for feature detection would be to use `require` instead of `import`:

```jsx
let InteractionManager;

try {
  InteractionManager = require('react-native').InteractionManager;
} catch (e) {
  // handle gracefully
}
```

But this breaks bundlers such as Vite which don't handle mixed ESM and CJS by default. Making any such change in a library "breaking".

Because of different behavior across bundlers and configurations, it makes it non-trivial to handle such scenarios. Currently we opted in to use `.native` extension for `require` in try/catch, but I'm not confident that it won't break for some other configuration. Frameworks such as [One](https://onestack.dev/) use Vite as bundler instead of Metro, and many apps use [Repack](https://re-pack.dev/) which uses Rspack, so there is precedence for other bundlers.

We recently encountered this issue in React Navigation. React Navigation 7 supports React Native 0.72+, so we need to gracefully handle this to not break apps that are still using older versions of React Native with support for `InteractionManager`, and currently maybe in process of migrating away, but not ready (e.g. [Expensify](https://github.com/Expensify/App)).

## Detailed design

This is more meant for discussion than concrete design proposal, but I thought of a few options that could be considered:

1. **Add a shim that throws on access/usage instead of throwing in getter**. There could even be a re-usable proxy that throws on any property access, so we don't need to create separate shims for each removed module.
2. **Remove the export and don't throw at all**. The import isn't present anymore, so it would be a hard error either way. A simple search should surface that the API was removed.
3. **Make the getter non-enumerable** (Untested whether this will avoid eager access, tho this might break other things)

I'd appreciate other ideas.

Most of the above applies to warning in getters as well, but they aren't as bad since warnings can be silenced. But I think it's worth re-considering how the warnings are handled well.

## Drawbacks

With a shim, the main tradeoff is that developers would discover the removal only when the API is used. A shim would also remain truthy, so checking whether the export exists wouldn't establish whether it's usable.

## Alternatives

The detailed design section outlines multiple approaches, so I'm leaving this empty for now.

## Adoption strategy

Once an approach is agreed on, we should apply it to future API removals and consider updating existing throwing getters.

## How we teach this

The React Native documentation should describe the removal process and provide a recommended pattern for libraries that support versions both before and after an API's removal.

Release notes should identify the removed API, its replacement, and any migration steps.

## Unresolved questions

Which approach should we use? Should the same rules apply to deprecation warnings?
