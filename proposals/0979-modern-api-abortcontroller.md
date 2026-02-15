---
title: Align `AbortController` and `AbortSignal` with modern whatwg/dom spec
author:
- retyui
date: today
---

# RFC0979: Align `AbortController` and `AbortSignal` with modern whatwg/dom spec

## Summary

React Native use 7 years old package [`abort-controller`](https://www.npmjs.com/package/abort-controller) to polyfill `AbortController` & `AbortSignal`, see: https://github.com/facebook/react-native/blob/20779edd3bc9f1c205a063e15b324466ee509c24/packages/react-native/Libraries/Core/setUpXHR.js#L37-L44

Since then, whatwg/dom spec has evolved and added new APIs, such as `AbortSignal.timeout()`, `AbortSignal.any()`, `AbortSignal::reason`, etc.


## Basic example

```tsx
fetch(url, { signal: AbortSignal.timeout(30_000) });
//                             ^^^ Crash: TypeError: AbortSignal.timeout is not a function (it is undefined)
```

## Motivation

- no extra polyfills needed: out of the box implementation of `AbortController` and `AbortSignal` is very old and doesn't support modern APIs, which forces users to install their own polyfill
- reduce app size: the `abort-controller` depends on `event-target-shim` which is duplicate! As React Native has its own implementation of `EventTarget` (see: `packages/react-native/src/private/webapis/dom/events/EventTarget.js`)
- simplified syntax: the modern API is more ergonomic and easier to use, for example, `AbortSignal.timeout()` is much easier to use than creating a new `AbortController` and calling `setTimeout` manually
- full control over implementation: by implementing our own `AbortController` and `AbortSignal`, we can ensure that it is fully compatible with React Native and allow easily fix bugs and add new features in the future

## Detailed design

1. Remove `abort-controller` dependency and implement `AbortController` and `AbortSignal` in React Native codebase, aligned with modern whatwg/dom spec reusing existing `EventTarget` implementation
2. Add new APIs, such as `AbortSignal.timeout()`, `AbortSignal.any()`, `AbortSignal::reason`, etc.
3. Update existing TypeScript types to reflect the new implementation and APIs
3. Add Unit tests and Type tests for the new implementation and APIs
4. Update documentation to reflect the new implementation and APIs (https://reactnative.dev/docs/global-AbortController, https://reactnative.dev/docs/global-AbortSignal)

## Drawbacks

- Implementation cost: implementing our own `AbortController` and `AbortSignal` will require some development effort
- Potential for bugs: testing have to be thorough to ensure that the new implementation is fully compatible with React Native and doesn't introduce any new bugs
- Switching from `event-target-shim` to own `EventTarget` impl. may cause some unexpected issues

## Alternatives

- Use a different polyfill that is more up-to-date with the whatwg/dom spec
- Implement only the missing APIs on top of the existing `abort-controller` fork
- Do nothing and let users install their own polyfill if they need the modern APIs

## Adoption strategy

- No breaking change, as the existing APIs will still be available and work as before

## How we teach this

- Devs can use MDN Web Docs to learn about the modern APIs, as they are aligned with the whatwg/dom spec
