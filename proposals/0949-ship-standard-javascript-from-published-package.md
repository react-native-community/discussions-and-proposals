---
title: Ship Standard JavaScript from React Native Published Package
author:
- Bruno Bodian
date: 2025-10-20
---

# RFC0949: Ship Standard JavaScript from Published `react-native` Package

## Summary

This proposal seeks to modernize the `react-native` npm package by distributing standard JavaScript output (published `.js` files that contain no Flow syntax). This will help the ecosystem by significantly expanding compatibility with modern JS parsers and tooling, and reducing coupling with the user-land `@react-native/babel-preset`.

For Flow users, we will introduce a `"flow"` `exports` condition which points to the Flow source files.

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
// Standard JavaScript (no Flow syntax)
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

**Flow source (resolved via the `"flow"` `exports` condition):**
```javascript
// react-native/Libraries/Components/ActivityIndicator/ActivityIndicator.js
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

declare export function ActivityIndicator(
  props: ActivityIndicatorProps,
  ref?: React.Ref<HostComponent<empty>>
): React.Node;
```

## Motivation

### Why are we doing this?

The JavaScript tooling ecosystem is moving fast. There are very good alternatives, that are mainly rust and go-based, which are created to improve the build speed and performance. As of today, React Native still ships Flow code on npm, relying on userland babel to remove/transform Flow syntax even though most React Native developers don't use Flow.

### What problems does this solve?

#### 1. **Compatibility with Modern JavaScript Tooling**

Modern build tools and test runners increasingly do not support Flow syntax out of the box:
- **SWC and @swc/jest**: Fast Rust-based transpiler that is becoming the standard for performance-conscious projects. Flow syntax is not officially supported. They don't plan to support Flow https://github.com/swc-project/swc/issues/256.
- **esbuild**: Fast go-based bundler, doesn't plan to support Flow either. See https://github.com/evanw/esbuild/issues/79
- **oxc**: Fast Rust-based JavaScript toolchain.
- **Vitest**: Modern test runner that's gaining popularity as a Jest alternative.

Projects using these tools must add extra transformation layers specifically to handle Flow syntax from `react-native`, adding complexity and slowing build times.

#### 2. **Build Efficiency**

Currently, `react-native-babel-preset` removes/transforms Flow syntax on every build using a set of plugins (e.g. `@babel/plugin-transform-flow-strip-types` plus additional transforms such as `babel-plugin-transform-flow-enums`). This means every developer's build process repeatedly performs the same Flow-to-standard-JavaScript transformation. Doing this once at publish time would be more efficient and eliminate this redundant work from every build. Additionally, shipping standard JavaScript would open the door for React Native to potentially use bundlers other than Babel in the future.

#### 3. **Alignment with Ecosystem Standards**

- **Most major libraries ship plain JavaScript**: React and other major frameworks publish plain JavaScript with separate type definitions rather than type-annotated source code.

### What is the expected outcome?

After this change:
1. React Native will be **compatible with all modern JavaScript tooling** without additional configuration
2. React Native will be **future-proof** as the ecosystem continues evolving toward standard JavaScript
3. Build times and test times can be **significantly improved** through modern tooling adoption

## Detailed design

React Native's publishing pipeline would gain a Flow-to-standard-JavaScript transform step, and introduce a `"flow"` `exports` condition:

1. **Build Process**:
    - Use a publish-time transform pipeline by reusing the Flow-related transforms from `@react-native/babel-preset` to emit standard JavaScript output. This is not limited to stripping type annotations. It also includes transforms for Flow syntax features (e.g. enums).
    - Preserve the original Flow-typed sources, which will be available via the `"flow"` `exports` condition.

2. **Package Structure:**
    The npm package would include:
    - Plain `.js` files → standard JavaScript output (no Flow syntax)
    - Existing `.d.ts` files for TypeScript
    - Flow-typed source files for Flow users (via the `"flow"` `exports` condition)

3. **Source Repository:**  
   The React Native source would remain Flow-typed internally. Only the published output changes.

## Drawbacks

- **Build Pipeline**: Adds a Flow-to-standard-JavaScript transform step to the release process
- **Flow Users**: May perceive this as reduced Flow support (though the Flow-typed sources remain available via the `"flow"` `exports` condition)

## Alternatives

The alternative is to keep Flow syntax in JavaScript files and require each tool (SWC, esbuild, oxc etc.) to transpile Flow into standard JavaScript (strip type annotations and apply transforms for Flow-only syntax) using `flow-remove-types` plus additional transforms, or an equivalent pipeline (Rust-based for SWC and oxc, Go-based for esbuild etc). However, this would be a repetitive task executed each time these tools run, causing performance loss compared to shipping standard JavaScript.

## Adoption strategy

This is intended to be a non-breaking change for default consumers, and can be shipped in any release with no migration needed. Flow users can opt into the `"flow"` `exports` condition, which may require resolver/tooling configuration to select the `"flow"` condition.

## How we teach this

This change can be documented in the release notes.

## Unresolved questions

- Will the built files overwrite the source codebase locations on publish? Ideally not. This could involve copying more files into a dist/ directory, or using suffix-based filenames mapped via `exports`.
