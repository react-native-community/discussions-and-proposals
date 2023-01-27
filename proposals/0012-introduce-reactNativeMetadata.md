---
title: Introducing `reactNativeMetadata` to `package.json`, for RN specific metadata
author:
  - Lorenzo Sciandra
date: 25-01-2021
---

# RFC0012: Introducing `reactNativeMetadata` to `package.json`, for RN specific metadata

## Summary

This RFC wants to introduce and formally discuss a new section for the package.json of a react-native project (app or library), called `reactNativeMetadata`.

This new section will allow developer to express certain characteristics of their code in a more formalized manner, that can easily used by tooling to act towards the code accordingly.

**Note well**: this is but the first draft of the actual shape and fields. The goal of this draft is to collect feedback and help finalize the spec for this section.

## Basic examples

This is how this section would look like...

...for a library:

```json
{
    "name": "@rnx-kit/metro-serializer",
    "version": "1.0.11",
    ...,
    "reactNativeMetadata": {
        "version": "2.3",
        "type": "library",
        "features": {
            "fabric": "true",
            "android": {
                "turbomodules": "true",
            },
        },
        "requirements": {
            "react-native": ">= 0.71.0",
            "expo": ">= 46.0.0",
            "expo-modules-core": "1.2.0"
        }
    }
}
```

...for an app:

```json
{
    "name": "contoso",
    "version": "2.0.3",
    ...,
    "reactNativeMetadata": {
    "version": "1.0",
    "type": "app",
    "features": {
        "newArchEnabled": "true",
        "android": {
            "codegen": "true",
            "turbomodules": "true",
        },
        // Future example
        // "metroExportsMode": "strict"
    },
    ...
    }
```

## Motivation

The conversation that originated this idea was [an ask from the Meta team](https://github.com/microsoft/rnx-kit/issues/1863) to have `align-deps` help developers in the community migrating to the new architecture by providing insights into which libraries would have Fabric/TM support. Very quickly it became clear that there is currently no straightforward way to know if a library indeed has that support.

A similar problem was also encountered by [React Native Directory](https://reactnative.directory/), that tried to [programmatically infer](https://github.com/react-native-community/directory/pull/870) support via the presence of certain fields in the package.json of a library - this because one can get access to the manifest file via npm without having to download the entire node_module.

It became clear that there was a need to be able to communicate and parse certain information about a react-native library via its `package.json`, which led to [a meeting](https://github.com/microsoft/rnx-kit/discussions/2125) between Meta, Microsoft and Expo to figure out a common solution.

This RFC is the result of that meeting: a new section of the `package.json`, called `reactNativeMetadata`, to contain certain metadata about a react-native project - it being either a library or an app.

There are a few areas that this section could express in a consistent manner; here's a list of the ones we could think of (in no particular order):

- if a library supports the new architecture, turbo modules or fabric or both
- which versions of react-native a library supports in a given release
- if an app has opted in into Fabric, TM, or both
- if an app is using Hermes or another js engine

Sidenote: the name `reactNativeMetadata` was chosen to mirror the camel case pattern used by other fields such as `peerDependenciesMeta`.

## Detailed design

This is the section that in this first draft will need to be properly finalised. For now, here's what we envisioned:

```js
reactNativeMetadata: {
    version: string, // ex. 1.0.0 - we know that this spec might evolve in the future
    type: string; // "app" or "library"
    features: {
        // can be global...
        newArchEnabled: boolean,
        jsEngine: string, // ex. "hermes", "jsc", "V8"...
        // ...or platform specific
        android: {
            newArchEnabled: boolean,
            fabric: boolean,
            turbomodules: boolean,
            jsEngine: string,
        },
        iOs: {
        ... // same as android
        },
        macOs: {
        ... // same as android
        },
        windows: {
        ... // same as android
        },
        // more platforms, params - add a comment here
    },
    // this section would most likely only be relevant for libraries
    requirements: {
        "react-native": string, // semver compliant range of versions of RN supported
        expo: string, // semver compliant range of versions of Expo supported (if any)
    }
}
```

## Drawbacks

The main drawback is that we'd need to convince developers to start using it, so we'd have to expect a phase in which only a part of the userbase has this section configured and tool developers need to account for the section missing entirely.

## Alternatives

`reactNativeMetadata` is, by design, fully original and new: there are other files that are used to define configurations for various aspect of react-native based projects (such as `react-native.config.js`, `app.json`, Expo's `app.config.js`, `expo-module.config.json`) but this one does **not** overlap with any of them. Explicitly, this one has the purpose of being filled only with metadata to be read by package managers and other tools.

## Adoption strategy

Once the v1.0.0 version of the shape is finalised, we can start integrating it across the board in tools such has `align-deps` and the RNDirectory, and in the react-native tools themselves. This first wave of support would have to be coordinated around a certain release that lives in RN's main branch, such as now it'd be introduced in 0.72 (which branch has not been cut).

After adding this first wave of support, and related documentation, we could more broadly communicate to the community and the maintainers how adding this section would benefit them too. Especially for library maintainers we'll have another RFC coming up soon to describe how we are planning to better communicate this.

## How we teach this

- add to core's documentation (which should happen anyway if we start using it for flags for new arch configuration)
- have RNDirectory and align-deps mention a call to action to introduce this

## Unresolved questions

v1.0.0 of the shape needs to be finalized.
