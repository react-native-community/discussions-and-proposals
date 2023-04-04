---
title: Lazy Bundling
author:
- Moti Zilberman
date: 2022-04-03
---

# RFC0605: Lazy Bundling

## Summary

Automatically split development bundles at dynamic `import()` boundaries in order to speed up the loading of large apps.

## Motivation

In a large React Native app, the initial JavaScript bundle can be noticeably slow to build and load, as the time required scales with the amount of code in the project. By deferring the bundling of code loaded with `import()`, we can limit the amount of code that needs to be built for any given screen, making it independent of the total size of the app.

## Detailed design

### Changes to the core bundling algorithm in Metro

Without lazy bundling, Metro's core bundling algorithm consists of:

1. *Resolving* a module - starting with the *entry point* specified by the request.
2. *Transforming* the module with Babel, which also extracts the module's dependencies (`import` declarations, `require()` calls, `import()` calls).
    1. As an implementation detail, each module is wrapped in a [`define()`](https://github.com/facebook/metro/blob/c8da588a4498d3a8521bb0799143cfa539ed3b50/packages/metro-runtime/src/polyfills/require.js#L123) call.
    2. As an implementation detail, each `import()` call is compiled to a call to [`asyncRequire()`](https://facebook.github.io/metro/docs/configuration/#asyncrequiremodulepath).
3. Repeating resolution and transformation recursively for all the extracted dependencies.
4. *Serializing* all transformed modules:
    
    1. Assigning each module an ID.
    2. Injecting the IDs of dependencies into the `define()` call generated for each module.
    3. Concatenating all the `define()` calls into a bundle (with an optional source map).

Lazy bundling will modify steps (3) and (4) above so that:
* The bundler avoids traversing/transforming `import()` dependencies.
* For each `import()` dependency, the serializer adds the dependency's module ID and an *opaque bundle path* to its dependents' `define()` calls.
  * Conceptually, the bundle path may be any value that `__loadBundleAsync` (see below) knows how to handle.
  * As an implementation detail, the bundle path is a string, and contains a server-relative URL where the imported module can be fetched from Metro.
  * The serializer will add a `paths` object to the dependency map, containing a mapping from module IDs to bundle path.

As a result, the target of an `import()` dependency - along with its transitive dependencies - will not be included in the bundle, unless it's also the target of a static `import` or `require`.

> **NOTE:** Because lazy bundling does not change the transformer's output, cache entries from a lazy build can be reused in a full (non-lazy) build, and vice versa. This avoids penalizing projects that adopt a mix of lazy and non-lazy bundling.

### Changes to the Metro server

For backwards compatibility, lazy bundling will be **off by default in Metro**, and enabled by the `lazy=true` request parameter.

**React Native will enable lazy bundling by default** in development bundle requests (`dev=true`). Alternative bundlers used with React Native may ignore the `lazy=true` parameter and emit no calls to `__loadBundleAsync`.

Within the lifetime of a given Metro server instance, modules will receive stable IDs based on their paths. This will allow the client to skip evaluating copies of the same module that may occur across multiple bundles.

### `__loadBundleAsync` in Metro

The first time an `import()` call is evaluated with a given target, the default `asyncRequire()` implementation in Metro will call a new *framework-defined* global function named `__loadBundleAsync` with that target's bundle path (as produced by the serializer).

> **NOTE:** The `__loadBundleAsync` identifier will be prefixed with the currently configured [global prefix](https://github.com/facebook/metro/blob/69c8fc707bda418b4eb7aa646ad2887d83e1d3f1/packages/metro-config/src/defaults/index.js#L105), so the correct way to reference it at runtime is ``global[`${__METRO_GLOBAL_PREFIX__}__loadBundleAsync`]``. For simplicity we will continue to call it simply `__loadBundleAsync` in this RFC.

`__loadBundleAsync` must return a promise that resolves once the bundle has been fetched and evaluated (e.g. with `fetch` and `eval`).

```flow
// For lazy bundling: type SerializedBundlePath = string;
declare function __loadBundleAsync(path: SerializedBundlePath): Promise<void>;
```

`__loadBundleAsync` may be called multiple times with the same bundle path, including in parallel. It should implement caching as needed to avoid sending out unnecessary or duplicate requests. It is unspecified how `__loadBundleAsync` should derive its internal cache key from the provided bundle path.

If there is no `__loadBundleAsync` implementation available, the bundled code may throw a runtime error upon attempting to evaluate an `import()` call.

> **NOTE:** With the introduction of `__loadBundleAsync`, **we will deprecate the [`asyncRequireModulePath`](https://facebook.github.io/metro/docs/configuration/#asyncrequiremodulepath) option in Metro**. Providing a custom `__loadBundleAsync` implementation is expected to fulfil all current use cases for replacing `asyncRequire` at build time.

### `__loadBundleAsync` in React Native

In development builds, React Native will provide an implementation of `__loadBundleAsync` that fetches a bundle URL from the currently connected Metro server, integrates with Fast Refresh and LogBox, and provides feedback to the developer on the progress of loading a bundle.

### Relationship with Fast Refresh

Fast Refresh will continue to work as expected when multiple bundles are loaded. In particular, thanks to the feature's deep integration with Metro, it will remain possible to change static `import`s to dynamic `import()`s and vice versa with instant feedback.

### Relationship with bundle splitting

The Metro changes described above also *partially* enable more general support for bundle splitting, although that is not the main goal of this RFC.

Metro integrators may implement a custom bundle splitting solution (e.g. optimizing for chunk size) by:

1. Providing a custom serializer that generates custom bundle paths (not necessarily tied to a Metro server).
2. Providing a custom global implementation of `__loadBundleAsync` to process the bundle paths produced by (1).
3. Serializing multiple bundles from the same (fully resolved/traversed) dependency graph.

> **NOTE:** Documenting the experimental APIs in Metro for doing (1) and (3) is outside the scope of this RFC.

### Relationship with React Server Components

While React Server Components are not currently supported in React Native, they remain an area of interest for future development. The bundling infrastructure required to support Server Components has some overlap with what's required for lazy bundling. In particular, resolving a Client Reference in development would likely involve calling `__loadBundleAsync` on a bundle path returned from the server.

## Drawbacks

The key potential drawback is implementation complexity. However, the current proposal minimises this complexity by integrating lazy bundling into the core Metro algorithm (instead of creating a fork), and by keeping the integration surface between React Native and Metro as small as possible (encapsulated in `__loadBundleAsync` and `lazy=true`).

## Alternatives

This RFC is an evolution of an earlier implementation of lazy bundling at Meta, which:

* Used a non-Fast-Refresh-compatible mechanism for passing bundle paths from the serializer to `asyncRequire`.
* Did not include the `__loadBundleAsync` hook, instead closely coupling Metro to the React Native-specific way of fetching and evaluating bundles.
* Treated lazy bundling as a transform-time config setting instead of a request parameter, resulting in worse caching the inability to build production and development bundles using the same Metro config.

Apart from these implementation differences, the underlying concept remains the same as what we have been using at Meta since 2019, and we believe it is sound.

## Adoption strategy / How we teach this

Lazy bundling is not a breaking change from the perspective of React Native apps, so we can ship and enable it by default in the course of a single release of React Native. In development, all React Native apps already need to be connected to a working Metro server to support things like Fast Refresh, LogBox symbolication, and debugging - and this same requirement is sufficient for lazy bundling to work.

From user code's perspective the API for lazy bundling is `import()` + `React.lazy()`. These are APIs that already work in Metro (and in React web apps in general) so they do not need to be taught, but they are probably not used very often in existing React Native code (since they do not perform any form of bundle splitting by default).

To get the most benefit out of lazy bundling, developers will want to use a navigation library that uses `import()` as the mechanism for lazy-loading route components. We can work with leading libraries like React Navigation and Expo Router to move the ecosystem in this direction without requiring changes to user code.
 
Optionally, authors of navigation libraries may find it useful to ship AST transformations (Babel plugins) that inject `import()` calls at *all* route boundaries in development, regardless of whether the associated screen is intended to be lazy-loaded in production. This would provide a consistent set of split points and opt users of those libraries into a more scalable developer experience.

## Unresolved questions

* User-facing naming: Are we happy with "lazy bundling" as the name for this feature?
* APIs: Are we happy with `lazy=true` and `__loadBundleAsync` as the names for the new integration points between React Native and Metro?

## Acknowledgements

The [initial version of lazy bundling in Metro](https://github.com/facebook/metro/commit/72329d02f2513eec024c42aa777bd445f573a968) was implemented at Meta by @cpojer.
