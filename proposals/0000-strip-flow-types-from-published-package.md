---
title: Strip Flow Types from React Native Published Package
author:
- Bruno Bodian
date: 2025-10-19
---

# RFC0000: Strip Flow Types from React Native Published Package

## Summary

This proposal seeks to remove Flow type annotations from the JavaScript code published to npm in the `react-native` package. Currently, React Native ships JavaScript files that contain Flow annotations plus separate TypeScript definition files (`.d.ts`). This proposal would strip the Flow annotations from the JavaScript source files while maintaining the existing TypeScript definitions and adding Flow definition files (`.js.flow`) for Flow users.

## Basic example

**Current state** (what gets published to npm):

```javascript
// Current: Published with Flow annotations
import type {HostComponent} from '../../../src/private/types/HostComponent';
import type {ViewProps} from '../View/ViewPropTypes';
import StyleSheet, {type ColorValue} from '../../StyleSheet/StyleSheet';

type IndicatorSize = number | 'small' | 'large';

type ActivityIndicatorIOSProps = $ReadOnly<{
  hidesWhenStopped?: ?boolean,
}>;

export type ActivityIndicatorProps = $ReadOnly<{
  ...ViewProps,
  ...ActivityIndicatorIOSProps,
  animating?: ?boolean,
  color?: ?ColorValue,
  size?: ?IndicatorSize,
}>;

const ActivityIndicator: component(
  ref?: React.RefSetter<HostComponent<empty>>,
  ...props: ActivityIndicatorProps
) = ({
  ref: forwardedRef,
  animating = true,
  color = Platform.OS === 'ios' ? GRAY : null,
  // ... other props
}) => {
  // Component implementation
};
```

**Proposed state** (what would get published to npm):

```javascript
// react-native/Libraries/Components/ActivityIndicator/ActivityIndicator.js
// Plain JavaScript (Flow annotations stripped)
import StyleSheet from '../../StyleSheet/StyleSheet';
import Platform from '../../Utilities/Platform';
import View from '../View/View';

const ActivityIndicator = ({
  ref: forwardedRef,
  animating = true,
  color = Platform.OS === 'ios' ? GRAY : null,
  // ... other props
}) => {
  // Component implementation
};
```

**Flow definitions:**
```javascript
// react-native/Libraries/Components/ActivityIndicator/ActivityIndicator.js.flow
// @flow

import type {HostComponent} from '../../../src/private/types/HostComponent';
import type {ViewProps} from '../View/ViewPropTypes';
import type {ColorValue} from '../../StyleSheet/StyleSheet';

type IndicatorSize = number | 'small' | 'large';

type ActivityIndicatorIOSProps = $ReadOnly<{
  hidesWhenStopped?: ?boolean,
}>;

export type ActivityIndicatorProps = $ReadOnly<{
  ...ViewProps,
  ...ActivityIndicatorIOSProps,
  animating?: ?boolean,
  color?: ?ColorValue,
  size?: ?IndicatorSize,
}>;

/* Use canonical Flow syntax for the component */
declare export function ActivityIndicator(
  props: ActivityIndicatorProps,
  ref?: React.Ref<HostComponent<empty>>
): React.Node;
```

## Motivation

### Why are we doing this?

React Native currently ships JavaScript files that contain Flow annotations plus separate TypeScript definition files (`.d.ts`) to npm. While maintaining dual type support was pragmatic when the ecosystem was split between Flow and TypeScript, the Flow annotations embedded in the JavaScript source code have become a **significant barrier** to adopting modern JavaScript tooling, even though separate TypeScript definitions already exist.

### What problems does this solve?

#### 1. **Compatibility with Modern JavaScript Tooling**

Modern build tools and test runners increasingly do not support Flow syntax out of the box:
- **SWC and @swc/jest**: Fast Rust-based transpiler that is becoming the standard for performance-conscious projects. There is no official Flow support.
- **esbuild**: Ultra-fast go-based bundler.
- **Vitest**: Modern test runner that's gaining popularity as a Jest alternative.

Projects using these tools must add extra transformation layers specifically to handle Flow syntax from `react-native`, adding complexity and slowing build times.

#### 2. **Build Efficiency**

Currently, `react-native-babel-preset` strips Flow types on every build using `@babel/plugin-transform-flow-strip-types`. This means every developer's build process repeatedly performs the same Flow stripping operation. Stripping Flow types once at publish time would be more efficient and eliminate this redundant work from every build. Additionally, shipping plain JavaScript would open the door for React Native to potentially use bundlers other than Babel in the future.

#### 3. **Alignment with Ecosystem Standards**

- **Most major libraries ship plain JavaScript**: React and other major frameworks publish plain JavaScript with separate type definitions rather than type-annotated source code.

### What is the expected outcome?

After this change:
1. React Native will be **compatible with all modern JavaScript tooling** without additional configuration
2. React Native will be **future-proof** as the ecosystem continues evolving toward standard JavaScript
3. Build times and test times can be **significantly improved** through modern tooling adoption


## Detailed design (TBD)

Add a Flow stripping step to the React Native release process:

1. **Build Process**: Strip Flow annotations using `@babel/plugin-transform-flow-strip-types` before publishing
2. **Package Structure**: Publish plain JavaScript + existing TypeScript definitions + `.js.flow` files
3. **Source Repository**: Keep Flow annotations in source

## Drawbacks

- **Build Pipeline**: Adds a Flow stripping step to the release process
- **Flow Users**: May perceive this as reduced Flow support (though `.js.flow` files maintain compatibility)

## Alternatives

The only alternative is to keep Flow annotations in JavaScript files and require each tool (SWC, esbuild, etc.) to strip Flow types using `flow-remove-types` or equivalent (Rust-based for SWC, Go-based for esbuild etc). However, this would be a repetitive task executed each time these tools run, causing performance loss compared to shipping plain JavaScript.

## Adoption strategy

Since this is a non-breaking change (`.js.flow` files maintain Flow compatibility), it can be shipped in any release with no migration needed.

## How we teach this

This change can be documented in the release notes.

## Unresolved questions

- Should we publish `.js.flow` files for Flow users, or drop Flow support entirely?

