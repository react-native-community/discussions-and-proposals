---
title: Removing deep imports from react-native
author:
  - Alex Hunt <huntie@meta.com>
date: 2024-04-04
---

# RFC0894: Removing deep imports from `react-native`

## Summary

In this RFC, we propose removing **subpath imports** (or "**deep imports**") from React Native's JavaScript API.

These imports, such as the below, allow direct access to internal modules. While historically supported, we plan to phase out subpath imports to reduce our API surface area.

```ts
// Before - subpath imports allowed
import {Alert, AlertType} from 'react-native/Libraries/Alert/Alert';

// After - must import from 'react-native'
import {Alert, AlertType} from 'react-native';
```

Doing this will:

- Define a clear public API contract for `react-native` that minimises future breaking changes.
- Improve maintainability for React Native and its ecosystem.

#### Planned sequencing

- React Native 0.80: Deprecation of subpath imports, introduce runtime warnings.
- React Native 0.82 (or later): Removal of subpath imports via `"exports"` and the Strict TypeScript API.

Between deprecation and removal, we will hold an API consultation period over ***at least*** two releases.

## Motivation

We’re driven to remove subpath imports for these reasons:

- **Simpler and more stable API**: Limiting imports to one runtime entry point reduces reliance on unstable internal modules, giving us exact control over the public API.
- **Package encapsulation**: Again, keeps our internal structure flexible, letting us refactor without breaking user code.
- **Simplify React Native upgrades**.
- **Alignment with standards**: Modern JavaScript packages, including React, favour single entry point APIs, making React Native more consistent and intuitive.
- **Reduces standards conflicts**. (e.g. under [`package.json` `"exports"` support](https://metrobundler.dev/docs/package-exports/#breaking-import-specifiers-are-matched-exactly), we can't apply [platform-specific module](https://reactnative.dev/docs/platform-specific-code) resolution on subpaths.)

### Towards a stable React Native API

**Without a controlled package entry point, we cannot provide a stable React Native API.**

Our vision with React Native's Stable API effort is to simplify and better define how apps and frameworks consume React Native. Eventually, these changes will let us iterate towards a stable, long-term `react-native` package we aim to keep unbroken across releases.

## Detailed design

### Clarifying scope

When removing access to subpath imports, we refer to the *runtime API* of the `react-native` npm package.

- **Included**: JavaScript modules imported by an app or library, that are bundled by Metro and executed at runtime.
- **Not included**: Non-runtime support files such as scripts, e.g. `./cli.js`, `./jest-preset.js`, `./jest/`.

In practical terms, this means imports to modules under `react-native/Libraries/*` will be removed, in favour of the **main entry point** (`react-native/index.js`).

#### Removal of internal APIs is a breaking change

This move, shipped first as a non-breaking deprecation, will intentionally result in breaking changes for developers.

The current "public API" of `react-native` is unbounded, since any module, and any potentially internal, intermediary code can be accessed.

```ts
// Before - subpath imports allowed
import {Alert, AlertType} from 'react-native/Libraries/Alert/Alert';
import NativeAlertManager from 'react-native/Libraries/Alert/NativeAlertManager';

// After - must import from 'react-native'
import {Alert, AlertType} from 'react-native';
// - ✅ Alert and AlertType APIs are public
// - 🚫 Internal NativeAlertManager.js module is private and inaccessible
```

> [!Note]
> **💡 Most apps**, if following our documented APIs, should already be using the `'react-native'` import path exclusively.

### What will be in the public API?

By removing subpath imports, we will scope the `react-native` package down to a narrower, enumerable public API, using the existing `index.js` and `index.d.ts` exports as a starting point.

All existing APIs exported from the main `'react-native'` import path will be **unchanged**. As of March 2025, we now define a strict list of type exports, see [`index.js.flow`](https://github.com/facebook/react-native/blob/main/packages/react-native/index.js.flow) — symbols that are not in this file will not be part of the public API.

#### Framework considerations

One index file for React Native's public API is our ideal final state — for apps, libraries, and [React Native Frameworks](./0759-react-native-frameworks.md). However, we may revise this decision down the line if there emerges a strong need for an escape hatch for specific internals. The intent is for us to narrow from all files → explicit entry point(s) to our packages.

This will be open to input during the API consultation period, and we'll directly engage with Expo and our partners around this.

#### First party usages

We will also address any first party usages of subpath imports within React Native itself — making sure these APIs are exposed on the main module.

Already actioned:

- `CodegenTypes`, `codegenNativeComponent`, `codegenNativeCommands` (https://github.com/facebook/react-native/pull/49854).

### Performance considerations

Routing all module imports through one index file has potential performance implications. We believe these will be negligible.

- **Existing index optimisations unchanged**: The shape of `index.js` is unchanged, preserving the existing inline `require()` pattern for internal imports. Metro will correctly optimise these imports in the production bundle.
    - (Shakeable ESM imports, which do not rely on this special pattern, are something we want to pursue in Metro down the line.)

## Adoption strategy

The impact of these changes is potentially considerable, particularly for React Native libraries and Frameworks. After an initial deprecation, we will take our time and listen to the ecosystem before shipping a hard removal.

- **Deprecate in 0.80**, with an API consultation period of at least two major releases.
- **Offer tools**: Provide migration aids, including dev-mode warnings, an ESLint rule, and via related TypeScript changes.
- **Fully remove when confident**.

### Deprecation period

#### Dedicated announcement post

We'll communicate the impact of this change to users in the 0.80 release post on the React Native website, as well as a dedicated deep-dive post.

#### Dedicated GitHub discussion thread

As well as monitoring new GitHub issues, this thread (maintained by [@huntie](https://github.com/huntie)) will form the main public communication channel during our API consultation period.

https://github.com/react-native-community/discussions-and-proposals/discussions/893

#### Lint and runtime warnings in 0.80

We aim to ship development-time warnings both in ESLint (surfacing in text editors/command line) and via a dev-only Metro transform plugin (surfacing in the JavaScript console).

```ts
import {Alert} from 'react-native/Libraries/Alert/Alert';
//      ^ Warning: Deep imports from React Native are deprecated.
//        'Alert' should be imported from 'react-native'.
//        Autofix:
//          import {Alert} from 'react-native';
```

**Under the hood**: As we are unsure about the % usage of [@react-native/eslint-config](https://www.npmjs.com/package/@react-native/eslint-config), we believe integrating via Metro is important for consistent visibility.

#### Removing subpaths from our TypeScript API (opt-in)

See [Parallel effort: Strict TypeScript API](#parallel-effort-strict-typescript-api).

### Eventual strong enforcement

Again, it's important that we give the community sufficient time to adapt to this change before we action a hard removal — we anticipate this will be during **H2 2025**.

Once the API consultation period is over, and we have reasonable confidence from libraries and partners, we'll action a hard removal of subpath imports via the [`package.json` `"exports"` spec](https://nodejs.org/docs/latest-v18.x/api/packages.html#exports).

**Future `react-native` package.json**

This will functionally disallow unlisted subpath imports under TypeScript, Metro, Jest, Node.js, and others.

```json5
{
  "exports": {  
    ".": "./index.js", // 🚫 Subpath imports disallowed
    // ... (non-runtime exports)
  }
}
```

---

## Parallel effort: Strict TypeScript API

> 🟢 In active development as of January 2025.

#### Discouraging use by removing subpaths under TypeScript

A parallel part of our adoption strategy is a new version of React Native's TypeScript API, auto-translated from the source codebase in Flow. While a separate effort, this forms an incentivising mechanism for removing subpath imports, on top of console and ESLint warnings.

This new API will no longer expose internal `react-native` modules under TypeScript — resulting in type errors against existing uses, and preventing new deep imports which may have occurred from auto-importing in the past.

#### User opt-in

By default, TypeScript will continue to use the existing manually defined types included in the `react-native` package, which allow subpaths.

When we ship the new Strict TypeScript API, it will be available as a user opt-in — by specifying the `"react-native-strict-api"` condition in a project's `tsconfig.json`.

```json
{
  "compilerOptions": {
    "customConditions": ["react-native-strict-api"]
  }
}
```

<small>See [TSConfig Reference - `customConditions`](https://www.typescriptlang.org/tsconfig/#customConditions).</small>

Under the hood, this opt-in will be implemented in React Native via a lenient `package.json` `"exports"` mapping, exposing only `"." → "types_generated/index.d.ts"` under this condition.

```json5
{
  "exports": {  
    ".": {
      "react-native-strict-api": "./types_generated/index.d.ts",
      "default": "./index.js"
    },
    "./*": {
      "react-native-strict-api": null, // ← 🚫 Subpath imports disallowed
      "default": "./*.js"
    }
  }
}
```


- The scope of this opt-in will be for the immediately analysed TypeScript project root, and will have no impact at runtime or on dependent projects.
- Having an user opt-in will be necessary both for subpath import removal, as well as breaking changes to some type shapes now generated from source (increasing correctness, consistency, and ergonomics).
- As with subpath import deprecation, we may consider an escape hatch for Frameworks, based on feedback.

#### Final state (coordinated removal of subpath imports)

The final state (potentially with a preceding opt-out state) will be a hard rollout of the new types in a future version of React Native, aligned with when we remove subpath imports in our `"exports"` mapping.

```json5
{
  "exports": {  
    ".": {
      "types": "./types_generated/index.d.ts",
      "default": "./index.js"
    }
  }
}
```
