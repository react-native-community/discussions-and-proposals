---
title: Supporting `import.meta`
author:
- Rob Hogan
date: 2025-04-28
---

# RFC0895: Supporting `import.meta`

## Summary

Implementation of [`import.meta`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta) in transformed source modules within JavaScript bundles, with per-module, host-defined properties.

## Motivation

`import.meta` is [part of the ECMAScript specification](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-meta-properties) and is used in ES Modules across the ecosystem.

When ES Modules are bundled into a plain script, an untransformed `import.meta` expression inherently appears outside of the context of an ES Module, and engines will throw (typically `SyntaxError`) on evaluation. This represents a failure of the bundle to faithfully mirror the behaviour of valid input sources.

Practically, the motivation to support this has increased since React Native 0.79, which enabled `package.json#exports` field resolution by default. That led to bundling ESM versions of `node_modules` packages where previously Metro would resolve and bundle CJS versions. Such failures are difficult for the user to action, requiring custom resolvers, upstream changes, or workarounds like `patch-package`.

## Detailed design

### Transforming `import.meta`

Metro will transform `import.meta` so that engines will not throw on evaluation, and we may define the behaviour.  

As an implementation detail, each `import.meta` (Babel `MetaProperty` nodes) will be transformed to a `MemberExpression` on the `module` pseudo-global, i.e. `module.importMeta`. References to `module.importMeta` in untransformed source will be prohibited.

### `import.meta` at runtime - calling `__getImportMetaProperties`

The first time `import.meta` (or rather, its transformed `module.importMeta` - we will continue to refer to `import.meta` as it appears in source) is accessed within a module implementation, Metro's runtime will create it as a writable, null-prototype object, and optionally call a new, *framework-defined* global function named `__getImportMetaProperties`.

If `__getImportMetaProperties` is nullish, `import.meta` will remain defined but have no properties.

Otherwise, `__getImportMetaProperties` will be called with information about the module and must return an object whose enumerable properties will be copied to `import.meta`.

```flow
type Module = $ReadOnly<{id: number}>; // Properties TBD
declare function __getImportMetaProperties(module: Module): {...};
```

`__getImportMetaProperties` will be called at most once in the lifetime of a module, and, as the specification allows, properties of `import.meta` may subsequently be added to, unset or overwritten by the module itself.

> **NOTE:** The `__getImportMetaProperties` identifier will be prefixed with the currently configured [global prefix](https://github.com/facebook/metro/blob/v0.82.2/packages/metro-config/src/defaults/index.js#L107), so the correct way to reference it at runtime is ``global[`${__METRO_GLOBAL_PREFIX__}__getImportMetaProperties`]``. For simplicity we will continue to call it simply `__getImportMetaProperties` in this RFC.

### `__getImportMetaProperties` in React Native

React Native will provide an implementation of `__getImportMetaProperties` that defines `url`

## Drawbacks


## Alternatives


## Adoption strategy / How we teach this


## Unresolved questions

* APIs: What should be the shape of `Module`  

## Acknowledgements

