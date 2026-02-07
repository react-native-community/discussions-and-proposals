---
title: 3P Extensions API for React Native DevTools panels
author:
- Alex Hunt
date: 2026-02-07
---

# RFC0978: 3P Extensions API for React Native DevTools panels

## Summary

A public API enabling third-party extensions to register custom DevTools panels in React Native DevTools. This RFC covers two design pieces for open source: the extension loading API in `@react-native/dev-middleware`, and the extension author contract (package format and manifest). Internal implementation details of RNDT are out of scope.

## Basic example

An integrator (e.g. Expo) configures extensions to load via `createDevMiddleware()`:

```js
createDevMiddleware({
  serverBaseUrl: 'http://localhost:8081',
  unstable_extensionsConfig: {
    extensions: [
      {
        name: 'My Extension',
        packageName: '@company/devtools-extension',
        enabled: true,
        basePath: '/absolute/path/to/node_modules/@company/devtools-extension',
        devtoolsPage: 'devtools.html',
        manifestPath: 'manifest.json',
      },
    ],
  },
});
```

An extension author ships a package containing a `manifest.json` and a `devtools.html` entry point — following the same shape as a Chrome DevTools extension, but with a strictly reduced subset of capabilities.

## Motivation

**Why now?** React Native DevTools shipped with 0.76 as a purpose-built debugging experience, but does not currently support third-party extensions. There is strong demand from multiple directions:

- **Open source community projects** are already building extensions systems independently. [Rozenite](https://github.com/callstackincubator/rozenite) (by Callstack) is a plugin framework that adds custom debugging panels to RNDT, with plugins for Redux DevTools, TanStack Query, and others. [Expo Dev Plugins](https://docs.expo.dev/debugging/devtools-plugins/) provide an out-of-tree mechanism for loading DevTools plugins (React Navigation, Apollo Client, TanStack Query, Redux, Async Storage) into Expo's debugging workflow. Both projects demonstrate real user demand and validate the design space — but they each solve the problem in a non-standard way.
- **User demand** for Redux DevTools, Relay DevTools, and other ecosystem tooling in RNDT is one of the most requested features since the 0.76 launch (see [community discussion #819](https://github.com/react-native-community/discussions-and-proposals/discussions/819)).
- **Internal demand within Meta** for custom panels in RNDT to support internal tooling and workflows.

A first-party API would unify these efforts behind a single contract, reduce fragmentation, and allow the broader ecosystem to build on a stable foundation.

## Detailed design

### Guiding principle

> React Native DevTools Extensions are a **strictly reduced subset** of Chrome's extension capabilities, but follow the **same API shape** as far as possible.

This means extension authors familiar with Chrome DevTools extensions will find the API familiar, and extensions can (in the subset we support) be tested in a Chrome environment. It also means we do not invent new concepts where Chrome already has them.

This RFC is **not** covering internal RNDT implementation details — only the `dev-middleware` API surface and the extension authoring contract.

---

### 1. Extension config API (for integrators)

Embedders configure extensions as part of the options to `createDevMiddleware()` in `@react-native/dev-middleware`. This allows the embedder (e.g. Expo, via Expo Dev Plugins and `app.json`) to manage how extensions are discovered and configured by the user. The system loads extensions into RNDT given their packages are present on the filesystem.

#### `createDevMiddleware()` options

```js
type Options = {
  serverBaseUrl: string | URL,
  // ...existing options...

  /**
   * Configuration for loading DevTools extensions.
   * Unstable API with no semver guarantees.
   */
  unstable_extensionsConfig?: ExtensionsConfig,
};
```

#### `ExtensionsConfig`

```js
type ExtensionDefinition = Readonly<{
  /** Display name. */
  name: string,

  /** Extension package name or other unique identifier. */
  packageName: string,

  /** Whether the extension is enabled. */
  enabled: boolean,

  /** Absolute path to extension package root. */
  basePath: string,

  /** Path to devtools page (entry point) within package (relative to package root). */
  devtoolsPage: string,

  /** Path to manifest.json within package (relative to package root). */
  manifestPath: string,
}>;

type ExtensionsConfig = {
  extensions?: ReadonlyArray<ExtensionDefinition>,
};
```

At runtime, `dev-middleware` resolves each package on the filesystem, validates the manifest, and serves the extension assets from the development server.

- `/debugger-frontend/embedder-static/extensionsConfig.js` — dynamically served script (loaded by the RNDT frontend) that populates `globalThis.__DEVTOOLS_EXTENSIONS__` with the resolved extension entries.
- `/devtools-extensions/<packageNameSanitized>/<asset>` — serves extension static assets.

---

### 2. Extension author contract

An extension is a package containing static assets served by `dev-middleware`. The package structure follows Chrome's extension model.

#### Minimal package structure

```
my-devtools-extension/
├── manifest.json
├── devtools.html
├── devtools.js
└── panel.html
```

#### `manifest.json` (supported subset)

Only the following fields from Chrome's `manifest.json` are supported. Other fields will be ignored (or rejected at validation time — TBD).

```json
{
  "manifest_version": 3,
  "name": "My Extension",
  "version": "1.0.0",
  "devtools_page": "devtools.html"
}
```

| Field | Required | Description |
|---|---|---|
| `manifest_version` | Yes | `2` or `3` |
| `name` | Yes | Display name |
| `version` | Yes | Semver version string |
| `devtools_page` | Yes | Path to the DevTools entry point HTML page |

Fields like `background`, `content_scripts`, `permissions`, `web_accessible_resources`, and `content_security_policy` are **not supported** in this initial version.

#### Bootstrap code (`devtools.html` / `devtools.js`)

The DevTools entry point must include a bootstrap snippet to receive the extension API via `postMessage`. This mirrors how Chrome injects the `chrome.devtools` API into extension iframes.

```html
<!-- devtools.html -->
<script src="devtools.js"></script>
```

```js
// devtools.js
window.addEventListener('message', function onAPI(event) {
  if (event.data?.type === 'extensionAPIInjection') {
    window.removeEventListener('message', onAPI);
    (0, eval)(event.data.script);
    // chrome.devtools.panels and chrome.devtools.inspectedWindow
    // are now available.
    main();
  }
});
window.parent.postMessage('requestExtensionAPI', '*');

function main() {
  chrome.devtools.panels.create(
    'My Panel',  // tab title
    '',           // icon path (unused)
    'panel.html', // panel page
  );
}
```

#### Panel page (`panel.html`)

The page registered via `chrome.devtools.panels.create()` is loaded inside an iframe in RNDT. It receives the same `chrome.devtools` API injection as `devtools.html`. Panels can communicate with the inspected React Native runtime via `chrome.devtools.inspectedWindow.eval()`:

```html
<!-- panel.html -->
<div id="root"></div>
<script>
  window.addEventListener('message', function onAPI(event) {
    if (event.data?.type === 'extensionAPIInjection') {
      window.removeEventListener('message', onAPI);
      (0, eval)(event.data.script);
      init();
    }
  });
  window.parent.postMessage('requestExtensionAPI', '*');

  function init() {
    // Evaluate JS in the inspected React Native runtime
    chrome.devtools.inspectedWindow.eval(
      'globalThis.__MY_EXTENSION_HOOK__?.getState()',
      function (result, isException) {
        document.getElementById('root').textContent =
          isException ? 'Error' : JSON.stringify(result, null, 2);
      },
    );
  }
</script>
```

#### Supported `chrome.devtools` APIs

The extension API injected at runtime exposes a strict subset of Chrome's `chrome.devtools` namespace. Initially:

- `chrome.devtools.panels.create(title, iconPath, pagePath)` — registers a new panel tab in RNDT.
- `chrome.devtools.inspectedWindow.eval(expression, callback)` — evaluates a JavaScript expression in the context of the inspected React Native runtime.

This is intentionally minimal. Further APIs can be added incrementally.

#### Optional: `@react-native/devtools-extensions-api` types package

We are considering publishing a types-only package, `@react-native/devtools-extensions-api`, that exposes TypeScript declarations for the strict subset of the `chrome.devtools` global API supported by RNDT. Extension authors would reference this in their `tsconfig.json`:

```json
{
  "compilerOptions": {
    "types": ["@react-native/devtools-extensions-api"]
  }
}
```

This provides type safety and editor autocompletion without requiring extension authors to pull in Chrome's full type definitions.

## Drawbacks

- **Maintenance cost**: Supporting a stable extension API adds a public API surface to maintain.
- **Subset limitations**: Extensions built for Chrome won't necessarily work in RNDT without modification, which may cause confusion.
- **Security surface**: Serving and executing third-party code in the DevTools frontend introduces a trust boundary. Mitigated by the fact that DevTools is a development-only tool.
    - We will (via CSP) only allow code to be loaded from the local dev server.

## Alternatives

- **Do nothing**: Let the community continue building independent solutions (Rozenite, Expo Dev Plugins). This leads to fragmentation.
- **Full Chrome extension compatibility**: Significantly higher implementation cost for capabilities (background scripts, content scripts, permissions) that aren't relevant to the DevTools panel use case.
- **Custom plugin protocol**: Invent a React Native-specific plugin system. This would lose the benefit of Chrome familiarity and existing ecosystem knowledge.

## Adoption strategy

This is a new, additive API — not a breaking change. Adoption path:

1. Integrators like Expo can adopt `unstable_extensionsConfig` to load extensions that are already distributed as npm packages.
2. Existing Expo Dev Plugins and Rozenite plugins can be migrated to the standard contract with minimal changes, given the similar shape.
3. The `unstable_` prefix signals that the API may evolve before stabilisation.

## How we teach this

The extension authoring contract deliberately mirrors Chrome's extension model, so documentation can reference Chrome's existing DevTools extension docs as a conceptual starting point, noting the reduced subset. A guide on the React Native website would cover the minimal package structure, bootstrap code, and supported APIs.

## Unresolved questions

- Should unsupported `manifest.json` fields be silently ignored or cause a validation error?
- Exact scope of `chrome.devtools.inspectedWindow` support in the initial version.
- Whether `@react-native/devtools-extensions-api` should be published as part of this proposal or deferred.
