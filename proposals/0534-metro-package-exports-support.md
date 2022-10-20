---
title: Package exports and strict entry points support in Metro
author:
- Alex Hunt <alexeh@meta.com>
date: 2022-10-17
---

# RFC0534: Package exports and strict entry points support in Metro

## Summary

To improve React Native’s compatibility with the npm package ecosystem, we intend to add support for the [`"exports"` field](https://nodejs.org/docs/latest-v18.x/api/packages.html#exports) in Metro. This feature was introduced in Node.js 12.7.0 and is described in the Node.js spec under [*Package entry points*](https://nodejs.org/docs/latest-v18.x/api/packages.html#package-entry-points).

## Basic example

The `"exports"` field is an expanded way for npm packages to define entry points within their package.json manifest. It is a modern alternative to `"main"` and the recommended method for exposing files in new packages.

```json
{
  "name": "my-package",
  "exports": {
    ".": "./lib/index.js",
    "./utils/*": "./utils/*.js",
    "./utils/*.js": "./utils/*.js",
    "./package.json": "./package.json"
  }
}
```

https://nodejs.org/docs/latest-v18.x/api/packages.html#package-entry-points

With `"exports"` support enabled, Metro will need to resolve paths appropriately based on what is defined by each package. We are proposing that package maintainers will be able to opt into strict handling of `"exports"`, with lenient and backwards compatible behaviour by default.

```js
import MyPackage from 'my-package';
// Loads ./lib/index.js

import foo from 'my-package/private/foo';
// Loads ./private/foo.js (Inaccessible under strict mode)
```

In addition to this example, `"exports"` introduces further behaviours which are discussed later in this proposal.

## Motivation

Developers using React Native should be able to use current JavaScript standards, conventions and packages to author their apps.

React Native developers expect a seamless experience when consuming packages from npm. Supporting `"exports"` within Metro has been a specific pain point ([facebook/metro#670](https://github.com/facebook/metro/issues/670)), with this feature having been adopted by other bundlers and a growing proportion of the JavaScript ecosystem.

The expected outcomes are that:

1. React Native projects will work with more npm packages out-of-the-box. Packages using `"exports"` will have associated non-breaking behaviours respected under Metro (described in the next section).
2. [React Native for Web](https://necolas.github.io/react-native-web/) projects will work with more npm packages out-of-the-box due to better-aligned resolution of the `"browser"` and `"react-native"` runtimes under `"exports"`.
3. We will provide a migration path for React Native (and other, willing) package maintainers to opt into strict and spec-compliant handling of `"exports"` under Metro, via a breaking release.
    - Headline benefit: Enforced encapsulation of private modules (and therefore reduced surface area for misuse/bugs).

## Priorities within this proposal

The Metro team intends to implement all features of `"exports"` described in this proposal. However, due to the size of the spec we are considering features in two priority levels.

- **P0 - Core features**: The robust subset of features that we must support in the spec for a minimum implementation of `"exports"`.
    - This work will be scheduled ahead of P1 features unless it makes sense not to during implementation.
    - App developers may wish to opt in to experimental `"exports"` support upon completion of these features.
- **P1 - Not targeted in first iteration**:
    - Attached to features which are more challenging to implement, or that were launched in later revisions to the Node.js spec.
    - These may still be done, otherwise will be fast-follow.
    - This work may be up for grabs by interested external contributors.

**All features discussed are P0 unless otherwise stated.**

## Detailed design

We intend to support all features of `"exports"` up to Node 18.10. For now, we are aware of but are not committing to support for [`"imports"`](https://nodejs.org/docs/latest-v18.x/api/packages.html#imports).

We are beginning in a state where `"exports"` is in general use and we cannot guarantee that references to these packages within existing React Native projects are compliant with the `"exports"` spec. Therefore Metro needs to carefully design around this.

The following features and considerations are detailed in this section:

- [Package encapsulation](#package-encapsulation)
- [Subpath exports](#subpath-exports)
- [Subpath patterns](#subpath-patterns)
- [Exact path specifiers](#exact-path-specifiers)
- [Conditional exports](#conditional-exports)
- [Conditional exports: `"import"` and `"require"`](#conditional-exports-import-and-require)
- [Conditional exports: Community definitions and `"browser"`](#conditional-exports-community-definitions-and-browser)
- [Conditional exports: User conditions](#conditional-exports-user-conditions)
- [Opting into strict `"exports"` handling](#opting-into-strict-exports-handling)

The following features will be implemented without any anticipated behaviour differences or special notes (links go to Node.js spec):

- [Exports sugar](https://nodejs.org/docs/latest-v18.x/api/packages.html#exports-sugar)
- [Self-referencing a package using its name](https://nodejs.org/docs/latest-v18.x/api/packages.html#self-referencing-a-package-using-its-name)

### Package encapsulation

This is the primary breaking change introduced by `"exports"`.

> When the `"exports"` field is defined, all subpaths of the package are encapsulated and no longer available to importers.\
– https://nodejs.org/docs/latest-v18.x/api/packages.html#main-entry-point-export

```js
"exports": {
  // Paths outside of this list are inaccessible to importers
  ".": "./lib/index.js",
  "./package.json": "./package.json"
}
```

**Proposed**:

- By default, Metro will ignore this constraint. We will log a warning when an inaccessible import is accessed.
- Under strict handling, Metro will prevent inaccessible imports and surface a resolution error to the developer.

### Subpath exports

This relates to the ability for packages to define subpath aliases that point to an underlying file.\
https://nodejs.org/docs/latest-v18.x/api/packages.html#subpath-exports

```json
"exports": {
  ".": "./index.js",
  "./submodule.js": "./src/submodule.js"
}
```

One edge case is a path conflict, for instance if the file tree for this package looked like this:

```
├── src
│   └── submodule.js
└── submodule.js
```

**Proposed**:

- Metro will consider all exact `"exports"` subpaths when present.
- In the case of a path conflict, when not using strict handling, Metro will prioritise the apparent file path over the explicit subpath if the file exists.

Illustration of subpath resolution for the above conflict example:

| | Default handling (non-strict) | Strict handling |
| - | - | - |
| `'pkg/submodule.js'` | `./submodule.js` | `./src/submodule.js` |
| `'pkg/src/submodule.js'` | `./src/submodule.js` | (Inaccessible) |
| | *Maintains compatible behaviour* | *Always reads from `"exports"`* |

### Subpath patterns

Subpath exports may use patterns to match several files. These can expose all contents of a directory, or include a [*pattern trailer*](https://github.com/nodejs/node/pull/39635) (e.g. file extension).\
https://nodejs.org/docs/latest-v18.x/api/packages.html#subpath-patterns

```json
"exports": {
  ".": "./index.js",
  "./utils/*": "./utils/*.js"
  "./utils/*.js": "./utils/*.js"
}
```

Subpath patterns are strictly a string replacement syntax. Therefore `*` acts as equivalently to a `**` file glob:

```js
import x from 'pkg/utils/x.js';
// Loads ./node_modules/pkg/src/utils/x.js

import y from 'pkg/utils/y/y.js';
// Loads ./node_modules/pkg/src/utils/y/y.js
```

Multiple `*`s cannot be matched in patterns ([source](https://github.com/nodejs/node/pull/39635#issue-958542966)):

```js
"./*/*.js": "./*.js" // Invalid subpath pattern means entry is ignored
```

**Proposed**:

- P1 (not targeted in first iteration): Metro will resolve subpath patterns.

This feature has a lower prioritisation due to possible implementation complexity and because the introduction of pattern trailers changed its behaviour more recently. The Node.js spec recommends explicitly listed imports for small packages (meaning we anticipate any fixes towards this will be weighted towards large packages).

> For packages with a small number of exports or imports, we recommend explicitly listing each exports subpath entry.\
— https://nodejs.org/docs/latest-v18.x/api/packages.html#subpath-patterns

### Exact path specifiers

> Package authors should provide either extensioned (`import 'pkg/subpath.js'`) [...] or extensionless (`import 'pkg/subpath'`) [...] subpaths [...] This ensures that there is only one subpath for each exported module so that all dependents import the same consistent specifier.\
— https://nodejs.org/docs/latest-v18.x/api/packages.html#extensions-in-subpaths

This detail has the potential to conflict with React Native's [platform-specific extensions](https://reactnative.dev/docs/platform-specific-code#platform-specific-extensions), if extensionless subpaths are not offered by the package, under strict handling. Metro's existing resolution behaviour (without `"exports"`) is as follows:

```js
// For extensions in `resolver.sourceExts`

import FooComponent from './FooComponent';
// Tries .[platform].js, .native.js, .js (+ TypeScript variants)

import BarComponent from './BarComponent.js';
// Tries suffixes following the extensioned filename,
// e.g. .js.[platform].js, .js.native.js, .js.js

// For extensions in `watcher.additionalExts`

import BazComponent from './BazComponent.mjs';
// Tries exact extension only
```

The second case above demonstrates less intuitive existing functionality within Metro, which we anticipate has not been in wide use. `"exports"` presents the opportunity to remove this.

**Proposed**:

- Keep current behaviour: Metro will try platform extension variants when resolving both extensionless and extensioned subpaths via `"exports"`.
- Under strict behaviour, Metro will treat extensioned imports as direct imports and will not attempt to resolve platform extensions.
- We will communicate to React Native package authors that extensionless exports should be preferred, since they allow the ability to introduce platform-specific modules where needed.

Not committed to in this RFC: We are considering deprecating or changing support for `.native.js` (raised in [facebook/metro#874](https://github.com/facebook/metro/pull/874)). This could be coupled with `"exports"` strict behaviour.

### Conditional exports

`"exports"` introduces a universal way for npm packages to specify alternative modules targeting specific platforms or environments.\
https://nodejs.org/docs/latest-v18.x/api/packages.html#conditional-exports

Conditions are provided via an object structure and can be nested. In this proposal, string identifiers for a conditional export are referred to as a ***condition name***, e.g. `"node"`, `"default"`.

```json
"exports": {
  ".": "./index.js",
  "./feature": {
    "node": "./feature-node.js",
    "default": "./feature.js"
  }
}
```

The Node.js spec documents two groups of condition names, which cut across multiple concepts.

- [Conditions implemented in Node.js core](https://nodejs.org/docs/latest-v18.x/api/packages.html#conditional-exports):
    - JavaScript environment, e.g. `"node"`, `"node-addons"`
    - Import method: e.g. `"import"`, `"require"`
- [Community conditions](https://nodejs.org/docs/latest-v18.x/api/packages.html#community-conditions-definitions):
    - Additional JavaScript environments, e.g. `"browser"`, `"deno"`
    - Build environment: e.g. `"development"`, `"production"`
    - (Others)

**Proposed**:

- Metro will implement resolution of all conditional exports within the core spec, and a subset of community defined conditions (described in the next two sections).
    - P1 (not targeted in first iteration): Support for nested conditions.
- We do not intend to use conditional exports as an alternative method to target the current React Native concept of platforms (e.g. `"android"`, `"ios"`, `"web"`). We will to continue to promote [platform-specific extensions](https://reactnative.dev/docs/platform-specific-code#platform-specific-extensions) for React Native package authors, where modules must be exposed as extensionless `"exports"` subpaths.
    - Special handling will be implemented for `"browser"`.

Our stance on keeping to the existing methods of targeting platforms is motivated by consistency of this feature between app and package developers, and not introducing additional concepts to the community.

### Conditional exports: `"import"` and `"require"`

`"import"` and `"require"` are listed among the core condition names supported by Node.js, intended to match modules depending on the syntax used to reference the module — either `import`/`import()` or `require()` (and variants) to select between CommonJS or ECMAScript modules.

For React Native projects, we have historically implemented no distinction between these module types, with packages assuming Babel-style module interop. Modules are converted on the fly via [@babel/plugin-transform-modules-commonjs](https://babeljs.io/docs/en/babel-plugin-transform-modules-commonjs), and Metro treats `import` and `require()` equivalently.

**Proposed**: Metro will provide no specific handling for `"import"` and `"require"` conditions. This support could come in future if we decide to move our module handling more tightly to the Node spec or there is a clear use case.

| Condition name | Node.js | Metro (proposed) |
| - | - | - |
| `"node"` | Always matched | - |
| `"node-addons"` | Matched when export requires native addons | - |
| `"default"` | Always matched | Always matched |
| `"import"` | Matched when loaded via `import` or `import()` | Always matched |
| `"require"` | Matched when loaded via `require()` | Always matched |

### Conditional exports: Community definitions and `"browser"`

Analysing the [Community Conditions Definitions](https://nodejs.org/docs/latest-v18.x/api/packages.html#community-conditions-definitions) documented in the Node.js spec and introduced previously:

- We note that `"browser"`, representing any web browser environment, is an established use case that will make sense to support in Metro — enabling packages to work seamlessly in React Native web apps.
- We see a space for `"react-native"`, representing a React Native runtime environment (i.e. Hermes or JavaScript Core VMs), to be introduced — enabling packages to make compatibility or performance decisions around this in select cases.

#### Current misalignment

Metro supports the [top-level `"browser"` field spec](https://github.com/defunctzombie/package-browser-field-spec) and a derivation of this, `"react-native"`. While our functionality around these fields is equivalent in concept to the proposed `"exports"` conditions, they differ in behaviour. We have the opportunity to replace these with more intuitive and community-aligned versions when translated to `"exports"`.

This can be seen in the defaults for Metro's [`resolverMainFields` option](https://facebook.github.io/metro/docs/configuration/#resolvermainfields) (applied by React Native CLI). As this is a static list of top-level fields, the resolver will invariantly prioritise `"react-native"` over `"browser"`:

```js
resolverMainFields: ['react-native', 'browser', 'main'],
```

We believe this default is non-ideal as it inhibits packages using the `"browser"` field spec from targeting react-native-web — instead working only for React Native-specific `.web.js` extensions. We believe the majority of non-React Native community package authors (in addition to React app developers) expect `"browser"` to used on web.

This default is directly misaligned with react-native-web via Webpack  ([docs](https://webpack.js.org/guides/package-exports/#target-environment)), which is used by a majority of users.

**Proposed**:

- **Breaking**: We will implement the `"browser"` condition name — changed from the top-level field to be selected when the React Native platform is `'web'`.
- **Breaking**: We will introduce a `"react-native"` condition name — changed from the top-level field to represent a native React Native JS runtime environment (and not web).
- Metro will not provide an implementation for other community conditions.

Combined, this is the only breaking change we foresee in Metro that will immediately impact app developers, and which we are not deferring via strict mode opt-in. It will align behaviour for existing users of react-native-web via Webpack.

| Condition name | Description | Node.js | Webpack | Metro (proposed) |
| - | - | - | - | - |
| `"browser"` | Any web browser environment | - | Matched when `platform === 'web'` | Matched when `platform === 'web'` |
| `"react-native"` [New] | Any React Native runtime environment (non-browser) | - | Matched when `platform !== 'web'` | Matched when `platform !== 'web'` |
| All other | | (unimplemented) | Various supported, e.g. `"deno"`, `"worker"`, `"electron"` | (unimplemented) |
| `"default"`| Makes no assumption about JS environment | Always matched | Always matched | Always matched |

Note: Now that `"react-native"` and `"browser"` will not overlap, packages intending to support both native and web environments but which want to expose alternative `"exports"` targeting *React Native **APIs*** should continue to use the `"default"` condition and use a [subpath export](#subpath-exports) instead: e.g. `import { reactNativeExport } from 'some-pkg/react-native'`.

### Conditional exports: User conditions

User conditions relates to the ability to set custom condition names within a project, which may be matched by target packages.

Prior art: [Webpack's `resolve.conditionNames`](https://webpack.js.org/configuration/resolve/#resolveconditionnames) and [Node.js' `--conditions` CLI argument](https://nodejs.org/api/cli.html#cli_c_condition_conditions_condition).

**Proposed**:

- P1 (not targeted in first iteration): We will provide a new config option within `metro.config.js` allowing app developers to define additional condition names which statically extend the default conditions matched.

```js
module.exports = {
  // ...
  resolver: {
    conditionNames: ['browser', 'production'],
  },
};
```

The exact name and shape of this option may change during implementation.

In addition, `resolver.resolveRequest` will continue to provide an escape hatch from Metro's handling of conditional exports, should an app need to override imports from a given package.

### Opting into strict `"exports"` handling

Since we are late to support `"exports"`, we cannot guarantee that enabling spec-compliant behaviour will not break existing users' apps. Even when we assume that packages have enabled `"exports"` via a breaking release, React Native projects may have updated to these under Metro's previous behaviour.

Therefore, Metro will default to lenient and backwards compatible behaviour, as has been noted in previous sections. We will provide an explicit opt-in method for strict handling, placing the responsibility for migration on React Native package authors. This allows these maintainers to have control over how their package behaves and to reconsider the exposed API under `"exports"`.

**Proposed:**

- Provide a strict mode opt-in for packages, enabling spec-compliant behaviour under Metro.
- Use non-strict handling by default, which will be a non-breaking change for app developers.

The new flag will be exposed as a `reactNative.metroExportsMode` field within `package.json`.

```js
{
  "name": "my-package",
  "exports": {...},
  // ...
  "reactNative": {
    "metroExportsMode": "strict"
  }
}
```

Note: `"reactNative"` (in camel case) will be a distinct field from `"react-native"` (in dash case, used as a `"main"` module alias that we are starting to deprecate with `"exports"`). We are [separately considering a `‌"newArchSupport"` option](https://github.com/microsoft/rnx-kit/issues/1863#issuecomment-1246580474) within this new top-level key, which will encapsulate options available to React Native packages.

#### Behaviour under `"nonstrict"` mode (default)

- Any subpaths are importable (package encapsulation is not enforced).
- When ambiguous, subpaths outside `"exports"` are preferred.
- `"exports"` can be mixed with top-level `"browser"` and `"react-native"` fields.
- **Breaking**: Conditions targeting `"react-native"` within `"exports"` will no longer match for react-native-web, unlike the previous `"react-native"` top-level field.
- All other features of `"exports"` are respected.

#### Behaviour under `"strict"` mode

- **Breaking**: Inaccessible subpaths are non-importable.
- **Breaking**: Platform-specific extensions will only be resolved for `"exports"` subpaths without file extensions.
- Metro will validate that `"exports"` cannot be mixed with top level `"browser"` or `"react-native"` fields.

## Adoption strategy

### Metro rollout

We plan to deliver `"exports"` functionality to app developers via a future major release of Metro once we consider P0 functionality to be stable. Before this point, we will ship an experimental Metro config option (likely, `resolver.experimentalPackageExports`) allowing app and package developers to preview functionality.

A major release will be warranted to deliver the indirect breaking change to the handling of `"browser"` and `"react-native"` conditions. However, all other features of `"exports"` at this stage will be lenient and backwards-compatible as described in *[Behaviour under `"nonstrict"` mode](#behaviour-under-nonstrict-mode-default)*.

We anticipate an easy upgrade for app developers — with changes only likely to affect the subset of developers using react-native-web with Metro (non-default) or standalone web projects with Metro. In addition to current handling in Webpack, Expo will have aligned on this new default before us ([expo/expo#19529](https://github.com/expo/expo/pull/19529)).

### Package adoption

We will recommend that packages should start adopting `"metroExportsMode": "strict"` ([described above](#opting-into-strict-exports-handling)) at the point of the next major React Native release that includes `"exports"`-ready Metro.

Package authors will then be able to drive this migration, reaching app developers in either minor or breaking package releases. We anticipate more breaking changes to materialise with these enforced changes to packages, such as from package encapsulation.

We anticipate adoption on this side to be gradual. `"exports"` and `"metroExportsMode": "strict"` will be recommended, but not mandated — see next section.

## How we teach this

### Adding `"react-native"` to the Node.js docs

As previously described, we will seek to include `"react-native"` in the list of [Community Conditions Definitions](https://nodejs.org/docs/latest-v18.x/api/packages.html#community-conditions-definitions) in the Node.js docs, increasing its discoverability for package authors across the npm ecosystem.

### Warnings in Metro Server

A visible place we can indicate incompatibilities with the `"exports"` spec between apps and packages is in Metro Server logs. This will inform app developers and help them towards updating their app code or nudging package authors. It is anticipated there will be a low occurrence of these edge cases in a typical React Native project.

- *Warning: You have imported the module "foo/private/fn.js" which is not listed in the "exports" of "foo". Consider updating your call site or asking the package maintainer(s) to expose this API.*

### Updating package.json boilerplate in template projects

We will update the following templates/tools used to scaffold React Native libraries to use `"exports"` and `‌"metroExportsMode": "strict"`:

- [create-react-native-library](https://www.npmjs.com/package/create-react-native-library)

### Metro/React Native website blog post

On completion, we will publish a new article on the availability of `"exports"` in React Native. This will include (or be wholly focused around) examples and recommendations to on updating packages to use `"exports"` as either a non-breaking or breaking release.

## Unresolved questions

### Removing the overlap of `"react-native"` and `"browser"`

#### Is this conceptually right?

Introducing the `"react-native"` condition name will be non-equivalent to the previous `"react-native"` top-level field (which is prioritised over `"browser"` and we are planning to invert). This hinges on:

- What does the community think currently when targeting `"react-native"`?
    - Assumption: *React Native for mobile or other non-web platforms*.
- Will the proposed treatment of `"react-native"` representing the React Native runtime (but not `react-native-web`) be clear enough?

#### Should we bring forward this change on the existing top-level fields?

As our current handling of the `"browser"` and `"react-native"` top-level is mismatched from the condition names we are introducing with `"exports"`, packages may end up with conflicting behaviour in the same `package.json`.

```js
{
  "main": "./index.js",
  "browser": "./index.browser.js", // Overridden by "react-native", even on web
  "react-native": "./index.react-native.js"
  "exports": {
    ".": {
      "browser": "./index.browser.js", // Changed: Used if platform == 'web'
      "react-native": "./index.react-native.js",
      "default": "./index.js"
    }
  }
}
```

We have the opportunity to tackle the alignment of this ahead of time to avoid confusion. We know that the existing behaviour is in most cases unhelpful. This could be actioned swiftly by changing defaults in React Native CLI and Expo Web (**update**: the latter is incoming in [expo/expo#19529](https://github.com/expo/expo/pull/19529)).

At minimum, we will describe in the docs that the handling of these keys within `"exports"` is fixed, whereas the top-level fields are configured per app (the `"react-native"` preference is set by React Native CLI, not Metro).

### Use cases for custom condition names

We are asking for input on the usefulness of this feature and use cases for apps, e.g. static `"development"` and `"production"` conditions.

If there is a clear case that projects will benefit from dynamic configuration of matched condition names (e.g. per platform), this will inform config design.

### Eventual deprecation of `"metroExportsMode": "strict"`

We anticipate that the majority of npm packages outside React Native libraries will not make use of this option. It will make sense to flip this default in a future version of Metro. Before this point however, we will likely introduce a config option for app developers to opt into "force strict" handling, given they see no `"exports"`-related Metro warnings for their project.

