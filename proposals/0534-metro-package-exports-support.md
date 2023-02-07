---
title: Package exports support in Metro
author:
- Alex Hunt <alexeh@meta.com>
date: 2022-10-17
---

# RFC0534: Package exports support in Metro

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

With `"exports"` support enabled, Metro will need to resolve paths appropriately based on what is defined by each package.

In this first iteration, we intend for `"exports"` paths to be handled inclusively on top of current resolution behaviour. In a future proposal, package maintainers will be able to opt into strict handling of `"exports"`.

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

We intend to initially roll out lenient and backwards compatible support for `"exports"` features. We posit that compatibility with `"exports"` packages is the most immediate unit of benefit we can provide to app developers, while keeping the surface area and upgrade guidance minimal for package maintainers.

The expected outcomes are that:

1. React Native projects will work with more npm packages out-of-the-box. Packages using `"exports"` will have associated non-breaking behaviours respected under Metro (described in the next section).
2. [React Native for Web](https://necolas.github.io/react-native-web/) projects will work with more npm packages out-of-the-box due to better-aligned resolution of the `"browser"` and `"react-native"` runtimes under `"exports"`.
3. Package maintainers will have less ambiguity and more control over which entry points are provided to React Native projects under `"exports"`.

At a later date, we will revisit strict mode in a separate proposal (initially drafted in earlier versions of this doc: [\#534](https://github.com/react-native-community/discussions-and-proposals/pull/534)).

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
- [Conditional exports: User conditions and configuration](#conditional-exports-user-conditions-and-configuration)
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

**Proposed**: Metro will ignore this constraint (reserved for a future strict mode). We will log a warning when an inaccessible import is accessed.

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

- **Breaking**: Metro will consider all exact `"exports"` subpaths when present. In the case of a path conflict, Metro will prioritise the file specified by `"exports"`.
    - If a subpath is specified in `"exports"` but fails to resolve (i.e. file doesn't exist), Metro will log a warning before falling back to filesystem resolution.

Illustration of subpath resolution for the above conflict example:

| Imported path | Current behaviour | Proposed behaviour |
| - | - | - |
| `'pkg/submodule.js'` | `./submodule.js` | `./src/submodule.js` |
| `'pkg/src/submodule.js'` | `./src/submodule.js` | `./src/submodule.js` |
| | | *Prioritises entries in `"exports"` (spec compliant)* |

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

This detail conflicts with React Native's [platform-specific extensions](https://reactnative.dev/docs/platform-specific-code#platform-specific-extensions). Metro's existing resolution behaviour around extensions is as follows:

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

- As noted in *Subpath patterns*, the `"exports"` spec allows subpaths to use pattern trailers (`*`) strictly for substitution. `"exports"` values must also be file paths with extensions. Therefore packages cannot express subpaths where the import specifier maps to a file with an expanded extension.

    ```js
    "exports": {
      "./subpath": "./subpath.js",
      "./subpath.js": "./subpath.js",
      "./subpath*": "./subpath*", // Insufficient to expand extension
    }
    ```

- The second example (e.g. `.js.js`) may be seen as unintuitive, given the emergence of extensioned JS imports. We have the opportunity to omit this logic when resolving `"exports"`.

**Proposed**:

- **Breaking**: Under `"exports"`, Metro will not resolve platform-specific extensions for listed package entry points.
    - When resolving any import specifier:
        - If the package defines `"exports"` and the import specifier (after expanding `sourceExts`) is matched, the package-defined path mapping will be used with no further transformation.
        - If there is no match in `"exports"`, Metro will look for files which match the import specifier, trying all extension variants (existing resolution logic).
- With this decision, we will have narrowed support for platform-specific extensions in packages. We will communicate to React Native package authors that alternative patterns should be used.
    - We have no near-term plans to drop platform-specific extensions for packages not using `"exports"`, or in app code.
- We will not take a strong stance on using extensionless or extensioned imports. The former may provide more flexibility for React Native package authors to change extensions in future without impacting consuming apps.

Note: We may yet (unplanned) independently make the platform-specific extensions feature work with extensioned specifiers, at which point this could be opened back up on `"exports"` entry points.

#### Illustrated

```json
"exports": {
  "./FooComponent": "./src/FooComponent.js",
  "./FooComponent.js": "./src/FooComponent.js",
}
```

Import specifiers listed in `"exports"` will be used when matched. Alternative paths will only be tried when there is no match in `"exports"`.

```js
import FooComponent from 'pkg/FooComponent';
// Reads from "exports":
//   pkg/src/FooComponent.js

import FooComponent from 'pkg/FooComponent.js';
// Reads from "exports":
//   pkg/src/FooComponent.js

import FooComponent from 'pkg/src/FooComponent';
// No match in "exports" (Metro will print warning)
// Tries files if present:
//   pkg/src/FooComponent.[platform].js
//   pkg/src/FooComponent.native.js
//   pkg/src/FooComponent.js
```

The last example given is not expected to enable backwards compatibility, but may serendipitously capture pre-existing imports in apps. In a future strict mode, paths outside `"exports"` will not be considered.

#### Workarounds

We will recommend that packages which rely on platform-specific extensions being available to consuming apps do not migrate to `"exports"` or update these entry points to handle platform internally.

One replacement pattern (besides `Platform.select()`) is a wrapper module:

```sh
└── src
    ├── FooComponent.js # Listed in "exports"
    ├── FooComponentImpl.android.js
    ├── FooComponentImpl.ios.js
    └── FooComponentImpl.js
```

```js
// pkg/src/FooComponent.js
export * from './FooComponentImpl'; // Will resolve platform exts
```

Conditional exports will be the recommended solution for replacing current `.native.js` entry points where aiming to identify React Native projects.

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
- We do not intend to use conditional exports as an alternative method to target the current React Native concept of platforms (e.g. `"android"`, `"ios"`, `"web"`).
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
- We see a space and community need for a `"react-native"` condition to be available under `"exports"`, replacing the existing `"react-native"` top-level field.

#### Improving current behaviour

Metro supports equivalent behaviour currently with the [top-level `"browser"` field spec](https://github.com/defunctzombie/package-browser-field-spec) and our custom top-level `"react-native"` field.

A pain point for developers has been the inability for packages to specify priority when using both of these fields, where the `"react-native"` entry point has historically been prioritised instead of `"browser"` in a react-native-web project. This (we believe) went against the general assumption of community package authors (in addition to React app developers) who intuitively expect `"browser"` to be used on web ([example](https://github.com/expo/router/issues/37)).

As we translate these conditions to `"exports"`, this can be improved due to the order-sensitivity of conditional exports — which moves control of which conditions are matched to package authors.

**Proposed**:

- We will implement the `"browser"` condition name. This will not be implicitly preferred over `"react-native"`, but will be read in the order specified by the package (per Node spec).
    - Metro will provide a built-in overridable default for this behaviour (see next section).
- We will introduce a `"react-native"` condition name representing all React Native projects (native and web). 
    - Metro will match this condition when configured for React Native (see next section).
- Metro will not provide an implementation for other community conditions.

| Condition name | Description | Node.js | Webpack | Metro (proposed) |
| - | - | - | - | - |
| `"browser"` | Any web browser environment | - | Matched when `platform === 'web'` | Matched when `platform === 'web'` |
| `"react-native"` [New] | Matched by the React Native framework (all  platforms) | - | Matched for React Native projects (when configured) (and/or "TBD" — [source](https://github.com/webpack/webpack.js.org/blob/65fa2e141b93b1fd4ef418ca85348c0ddebbd4e3/src/content/guides/package-exports.mdx?plain=1#L282-L294)) | Matched for React Native projects (when configured) |
| All other | | (unimplemented) | Various supported, e.g. `"deno"`, `"worker"`, `"electron"` | (unimplemented) |
| `"default"`| Makes no assumption about JS environment | Always matched | Always matched | Always matched |

Under this model, `"react-native"` and `"browser"` will continue to overlap. However, because earlier conditional exports entries have higher priority, package maintainers will have more fine-grained control over what is selected for their package — e.g. enabling matching of native platforms only when listed after `"browser"`.

Note: Packages intending to support both native and web environments but which want to expose alternative `"exports"` targeting *React Native **APIs*** should consider continuing to use the `"default"` condition and use a [subpath export](#subpath-exports) instead: e.g. `import { reactNativeExport } from 'some-pkg/react-native'`.

### Conditional exports: User conditions and configuration

User conditions relates to the ability to set custom condition names within a project, which may be matched by target packages.

Prior art: [Webpack's `resolve.conditionNames`](https://webpack.js.org/configuration/resolve/#resolveconditionnames) and [Node.js' `--conditions` CLI argument](https://nodejs.org/api/cli.html#cli_c_condition_conditions_condition).

**Proposed**:

- We will provide a new config option allowing app developers to define additional condition names which statically extend the default conditions matched.
    - A default value of `['react-native']` will be applied by React Native CLI ([as with the existing `resolverMainFields` and `platforms` options](https://github.com/huntie/react-native-cli/blob/3c80c062acb1d4a5490d32a28b5fb62cbe0abe64/packages/cli-plugin-metro/src/tools/loadMetroConfig.ts#L92-L93C60)).
- We will provide a new config option to assert certain condition names based on context, e.g. `platform`.
    - A default implementation matching `"browser"` when  `platform === 'web'` will be applied by metro-config.

```js
// metro.config.js
module.exports = {
  // ...
  resolver: {
    // The exports field condition names to assert globally
    // (default: ['react-native'])
    conditionNames: ['react-native', 'production'],

    // A function that will be used to assert additional
    // condition names when resolving an exports field path
    getAssertedConditions: ({ platform }) => {
      if (platform === 'web') {
        return ['browser'];
      }

      return [];
    },
  },
};
```

The exact name and shape of these options may change during implementation. Naming will ideally be broad enough to cover any future support for `"imports"`.

In addition, `resolver.resolveRequest` will continue to provide an escape hatch from Metro's handling of conditional exports, should an app need to override imports from a given package.

### Additional consideration: Jest

Because Metro's implementation of `"exports"` will not be strict, package resolution behaviour will be misaligned between Metro and Jest (where support was [added in Jest 28](https://jestjs.io/blog/2022/04/25/jest-28#packagejson-exports)). In projects consuming several exports-enabled dependencies, it is likely that misalignments will lead to confusing test failures.

Ideally, we can make use of the custom React Native test env introduced in [facebook/react-native#34971](https://github.com/facebook/react-native/pull/34971) to align major differences.

**Proposed:**

- Update [`customExportConditions `](https://jestjs.io/blog/2022/04/25/jest-28#packagejson-exports) in React Native's project template to include `"react-native"`.
- P1 (not targeted in first iteration): Align non-strict resolution in Jest via available configuration in React Native's Jest environment (potentially exposing a relevant integration point in metro-resolver).

## Adoption strategy

### Metro rollout

We plan to deliver `"exports"` functionality to app developers via a major release of Metro once we consider P0 functionality to be stable. Before this point, we will ship an experimental Metro config option (likely, `resolver.experimentalPackageExports`) allowing app and package developers to preview functionality.

We expect the breaking changes to be a manageable upgrade for app developers, impacting imports from the subset of packages which define `"exports"` and the subset of those with different pre-existing assumptions about `"exports"` handling in React Native.

- Removal of platform-specific extension handling when a subpath is present in `"exports"`:
    - **Medium risk**: Will impact cross-platform libraries where React Native is targeted via `.native.js` or where the top-level `"react-native"` field is expected to override `"exports"`.
    - **Low risk**: Will impact libraries using extensions to target individual platforms (`.[platform].js`). We assume the majority of packages which make use of this are React Native libraries, which are less likely to have migrated to `"exports"`.
- **Low risk**: Edge case conflicts between an aliased subpath defined in `"exports"` and a filesystem path.
- **Low risk**: Order-sensitivity of `"browser"` and `"react-native"` conditions under `"exports"` — may affect the subset of developers using react-native-web with Metro (non-default) or standalone web projects with Metro.

We will seek feedback from app developers who choose to preview `"exports"` functionality and aim to gather data about frequently-used packages before general rollout.

### Package adoption

We will recommend that packages should start adopting `"exports"` at the point of the next major React Native release that includes `"exports"`-ready Metro.

Package authors will then be able to drive this migration, reaching app developers in either minor or breaking package releases. While consuming projects with Metro will use non-strict handling we will recommend that most packages define a spec-ready list of exports rather than exporting all entry points.

We anticipate this adoption will be gradual. We will maintain long-term support for packages that don't switch to `"exports"`, for instance React Native-only libraries which do not want to drop platform-specific extensions.

## How we teach this

### Adding `"react-native"` to the Node.js docs

As previously described, we will seek to include `"react-native"` in the list of [Community Conditions Definitions](https://nodejs.org/docs/latest-v18.x/api/packages.html#community-conditions-definitions) in the Node.js docs, increasing its discoverability for package authors across the npm ecosystem.

**Update**: Submitted as [nodejs/node#45367](https://github.com/nodejs/node/pull/45367).

### Warnings in Metro Server

A visible place we can indicate incompatibilities with the `"exports"` spec between apps and packages is in Metro Server logs. This will inform app developers and help them towards updating their app code or nudging package authors. It is anticipated there will be a low occurrence of these edge cases in a typical React Native project.

- *Warning: You have imported the module "foo/private/fn.js" which is not listed in the "exports" of "foo". Consider updating your call site or asking the package maintainer(s) to expose this API.*

### Updating package.json boilerplate in template projects

We will update the following templates/tools used to scaffold React Native libraries to use `"exports"` over the `"main"` field:

- [create-react-native-library](https://www.npmjs.com/package/create-react-native-library)

### Metro/React Native website blog post

On completion, we will publish a new article on the availability of `"exports"` in React Native. This will include (or be wholly focused around) examples and recommendations to on updating packages to use `"exports"` as either a non-breaking or breaking release. The article may preview the future intended behaviour of opt-in strict handling, enabling package encapsulation and spec-compliant resolution.

## Unresolved questions

### Use cases for custom condition names

We are asking for input on the usefulness of this feature and use cases for apps, e.g. static `"development"` and `"production"` conditions.

If there is a clear case that projects will benefit from dynamic configuration of matched condition names (e.g. per platform or from other resolve-time info), this will inform config design.