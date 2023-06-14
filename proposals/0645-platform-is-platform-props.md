---
title: Platform isPlatform props
author:
  - Ryan Linton
date: 2023-06-14
---

# RFC0645: Platform isPlatform props

## Summary

Add `isAndroid`, `isIOS`, and `isNative` to `Platform`.

## Basic example

```ts
if (Platform.isAndroid) {
  // do some Android only stuff here
}
```

## Motivation

Platform already defines `isPad`, `isTV`, and `isTesting`. In consulting and the companies I've worked at I've seen dozens (even hundreds) of apps, and nearly all backfill the missing ones `isAndroid` and `isIOS` with something along the lines of the following:

```ts
const isAndroid = Platform.OS === "android";
const isIOS = Platform.OS === "ios";
```

`Platform` should provide these since so many devs want to express one-off platform checks this way.

## Detailed design

I did an initial implementation [here](https://github.com/facebook/react-native/pull/37870) but it has some drawbacks. As @NickGerleman [pointed out](https://github.com/facebook/react-native/pull/37870#pullrequestreview-1478897922) we don't want to maintain a centralized list of platforms. A better solution that keeps the platform definitions in their respective platforms is to update `Platform` to be a proxy object.

```ts
const Platform = new Proxy(
  {
    __constants: null,
    OS: "android",
    isAndroid: true,
    isNative: true,
    // ...
  },
  {
    get(target, prop) {
      if (prop in target) {
        return target[prop];
      }

      if (prop.toString().startsWith("is")) {
        return false;
      }

      return Reflect.get(...arguments);
    },
  }
);
```

## Drawbacks

- This change doesn't really offer any new functionality over conditional logic using `Platform.OS`. It's just a different (albeit quite common) way of expressing the same thing.
- As @motiz88 [points out](https://github.com/facebook/react-native/pull/37870#issuecomment-1591225470) this breaks existing optimizations with Babel transforms that remove dead code based on `Platform.OS` conditionals. This proposal likely further complicates that by making `Platform` a proxy object.
- All other platforms would need to update their `Platform` implementations to match.

## Alternatives

Another potential solution that doesn't use proxy but reads similarly would be to add an `is` method (e.g. `Platform.is('android')` or `Platform.is('ios')`). This may or may not have advantages considering the Babel transforms. It is less similar to the common code that inspired this proposal though.

## Adoption strategy

This is not a breaking change but would be a new API offering the same existing functionality provided by checking against `Platform.OS`. Existing React Native developers would upgrade to latest and discover via docs and types.

## How we teach this

Per motivation, React Native developers are already doing this. This just moves the definition of `isAndroid` and `isIOS` up into React Native.

## Unresolved questions

- Where are the Babel transforms for removing platform specific code and how do they work?
