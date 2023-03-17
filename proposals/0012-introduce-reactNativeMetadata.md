---
title: [DRAFT] Introducing `reactNativeManifest` to `package.json` for React Native specific metadata
author:
  - Lorenzo Sciandra
  - Nicola Corti
date: 25-01-2021
---

# RFC0588: [DRAFT] Introducing `reactNativeManifest` to `package.json` for React Native specific metadata

## Summary

> **Warning** ⚠️ This is the initial draft of this proposal. The goal of this draft is to collect feedback and help finalize the spec for this section.

This RFC introduces and formally discuss the concept of _React Native Manifest_, a set of metadata about a React Native project (either app or library). This metadata can be consumed by tooling to enhance the developer experience and support the React Native ecosystem.

Practically, this RFC proposes to add a `reactNativeManifest` section to the `package.json` file of a React Native app or library.
This new section will allow developers to follow a declarative approach to express capabilities of apps and libraries in a more formalized manner.

## Rationale

The need to specify **capabilities and metadata** for a react-native library is inherent to the React Native ecosystem.

As of today, there is no simple programmatic way to know if a NPM package is a React Native library. Good indicators could be:
* Presence of a `react-native:*` dependency in the `peerDependencies` section of the `package.json` ([example](https://github.com/RonRadtke/react-native-blob-util/blob/80628d418a5c81439dc2c1a1f05fae6406f0ba7f/package.json#L46C10-L49))
* Presence of React Native specific files such as `react-native.config.js` files
Those indicators are all non-exhaustive. 

Similarly, there is no easy way to understand if a library is exposing a native module or not. 
The React Native CLI is facing this difficulty and tries to circumvent it by inspecting the native code and search for subclasses of `*ReactPackage` using complex regular expressions ([example](https://github.com/react-native-community/cli/blob/605c542d18efcb02f217d3c85726fa73a79054c2/packages/cli-platform-android/src/config/findPackageClassName.ts#L35)) due to the lack of a declarative way to express this information.

Another practical examples of this need are the [React Native Directory](https://reactnative.directory/) attempting to [programmatically infer](https://github.com/react-native-community/directory/pull/870) New Architecture support from the presence of the `codegenConfig` fields in the `package.json` of a library. The `codegenConfig` filed is not intended to carry semantic information about New Architecture support, and is definitely a non-exhaustive solution for this problem.

While we can continue to build tooling that tries to infer metadata and capabilities of a library, providing a declarative way to declared such capabilites provides a series of benefit:
* **predictability**: developers can easily understand what a library is capable of doing and if such capability matches their app.
* **tooling**: tooling can easily consume this information and provide a better experience to developers.
* **single source of thruth**: a single manifest will act as a source of truth for all those capabilities and flag definition, that can easily be consumed by different platforms to provide a unified way to enable/disable capabilities.
* **verifiability**: the manifest can be easily verified & linted by tooling to ensure that app & library are not using an incompatible set of compatibilites. Similarly, the JSON schema we publish can be used to validate the `reactNativeManifest` section.

## Use Cases

This section lists the use case we envision for the `reactNativeManifest` section.

### Declaring the type of a project

For the sake of this RFC, we consider a React Native project as either an app or a library.
While an app project can contain native modules and components, it's **not** possible for a app to "consume" another app as a library as this would break the native build system invariants (i.e. a single `AndroidManifest.xml` and `Info.plist` file per app, etc.). If an app projects wishes to distribute some of its native modules or components, they need to be extracted into a library project.

We're proposing to declare if a project is either a library or an app by adding the `type` field (as `"app"` or `"library"`) to the `reactNativeManifest` section of the `package.json` file.

```json
{
  "reactNativeManifest": {
    "type": "library"
  }
}
```

### New Architecture support

One of the primary driver of this proposal is the [New React Native Architecture support](https://reactnative.dev/docs/the-new-architecture/landing-page). Specifically we believe that the ecosystem is currently affected by:
* Lack of a declarative way to define if a library is compatible with New Architecture
  * As described in the [rationale](#rationale), tooling needs to infer this information from the presence of the `codegenConfig` field in the `package.json` of a library. Libraries might decide to don't use codegen and still be compatible with New Architecture, so this method is not exhaustive.
  * The alternative at this stage is to inspect the code and check the API callsites if they are New Architecture or Old Architecture compatible.
* Lack of a declarative way to enable New Architecture on both platforms
  * Currently New Architecture is enabled by a Gradle Property `newArchEnabled` on Android in the `gradle.properties` file and by invokign `RCT_NEW_ARCH_ENABLED pod install` on iOS.
  * As of today, there is no way to enable New Architecture for an app project for both platforms
  * Moreover, the file vs environment variable leads to a scenario where you can't know if a app supports New Architecture or not by inspecting the code (as New Architecture support is known at  `pod install` time and is not codified in the codebase)

Therefore we propose to add the `newArch` section to the `reactNativeManifest.capabilities` of **both apps and libraries** with the following semantic:
* For Apps: `newArch.enabled==true` implies that the app wants to use the New Architecture.
* For Libraries: `newArch.enabled==true` implies that the library is compatible with the New Architecture.

Tools can be built on top of this information to check that a app with `newArch.enabled==true` is not accepting libraries with `newArch.enabled==false` and warning against library that don't have the key specified.

The setup would look as follows for both apps and libraries:

```json
{
  "reactNativeManifest": {
    "capabilities": {
      "newArch": {
        "enabled": true
      }
    }
  }
}
```

This section will allows also for split configuration between platforms:

```json
{
  "reactNativeManifest": {
    "android": {
      "capabilities": {
        "newArch": {
          "enabled": true
        }
      }
    },
    "ios": {
      "capabilities": {
        "newArch": {
          "enabled": true
        }
      }
    }
  }
}
```

#### `codegenConfig` support

Similarly to New Architecture support metadata, the `codegenConfig` is a key metadata of the New Architecture build pipeline.
Currently the `codegenConfig` is a **top-level** key in the `package.json` of a project. 

We propose to move this key under the `reactNativeManifest.capabilites` section as follows:
```json
{
  "reactNativeManifest": {
    "capabilities": {
      "codegenConfig": {
        "name": "MyLib",
        "type": "all",
        "jsSrcsDir": ".",
        "android": {
          "javaPackageName": "com.example.mypackage"
        }
      }
    }
  }
}
```

### Hermes Support

Similarly to New Architecture support, the current way to enable/disable the Hermes engine is toggled by using a Gradle Property `hermesEnabled` on Android and changing the `:hermes_enabled` property in the Podfile.

We propose to add the `hermes` section under the `reactNativeManifest.capabilities` section as follows:

```json
{
  "reactNativeManifest": {
    "capabilities": {
      "hermes": {
        "enabled": true
      }
    }
  }
}
```

which will also allow for split configuration between platforms as follows:

```json
{
  "reactNativeManifest": {
    "android": {
      "capabilities": {
        "hermes": {
          "enabled": true
        }
      }
    },
    "ios": {
      "capabilities": {
        "hermes": {
          "enabled": true
        }
      }
    }
  }
}
```

### React Native Release feature flagging

We currently have a numer of different files where capabilities of React Native can be toggled to enable/disable features in the runtime.
For example we have:
* [android/gradle.properties](https://github.com/facebook/react-native/blob/main/packages/react-native/template/android/gradle.properties) to specify key-value build time properties for Android
* [android/app/build.gradle](https://github.com/facebook/react-native/blob/main/packages/react-native/template/android/app/build.gradle) to specify build time configuration for Android
* [ReactNativeFeatureFlags.js](https://github.com/facebook/react-native/blob/main/packages/react-native/Libraries/ReactNative/ReactNativeFeatureFlags.js) which contains runtime feature toggles for the Javascript layer.
* [ReactFeatureFlags.java](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/config/ReactFeatureFlags.java) which contains runtime feature toggles for Android.
* [ReactNativeConfig.cpp](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactCommon/react/config/ReactNativeConfig.cpp) which contains runtime feature toggles for the C++ layer.

Most of those config flags are undocumented and they're not verified for invalid configuration. Specifically, some feature flags require to be toggled on multiple layers, and it's easy to forget to toggle one of them. Invalid combinations of those flags could lead to unexpected runtime behavior.

We propose to define `reactNativeManifest` as the **preferred entry point** for all the user facing entry points. All the tooling should be adapted to consume the information from the `reactNativeManifest` section and have sensible defaults defined by the tooling itself.

It's outside the scope of this RFC to declare all the new sections we intend to add, and we defer to the [Future proof and extensibility](#future-proof-and-extensibility) section for the process on how to include a new key to the `reactNativeManifest` section.

## Excluded use cases

This section lists the use cases where we believe the `reactNativeManifest` section should **not** be used.

### React Native version support

For the time being, we're not planning to add a `version` section to the `reactNativeManifest` section for libraries. The rationale is that we believe that the `react-native` version should be specified in either the `peerDependencies` or the `engines` section of the `package.json` of the project. This follows the idiomatic way to specify the version of a dependency in a NodeJS project.

### TurboModule/Fabric toggles

At the time of writing, we prefer to don't offer a dedicated section to toggle Fabric/TurboModule capability inside `reactNativeManifest`. The rationale is that we believe that the `newArch` section is sufficient to expose the New Architecture to users.
Selectively toggling Fabric/TurboModule is a more advanced feature. We believe we'll still be offering a more advanced way to feature toggle such infrastructure pieces, but at the current state, our preference is to not expose such capability in the top level `reactNativeManifest` section.

## Proposed Tooling

This section lists the tooling we believe could be built on top of `reactNativeManifest` to improve the developer experience.

### `align-deps` support

This RFC originated from a [conversation between Meta and Microsoft](https://github.com/microsoft/rnx-kit/issues/1863) to use `align-deps` as key feature in the New Architecture rollout support, in helping developers understanding if a library they're using is New Architecture compatible or not. Very quickly it became clear that there is currently no straightforward way to know if a library indeed has that support.

Therefore we believe `align-deps` can benefit from this information and be extended to offer New Architecture support information.

### React Native CLI support

Ideally the CLI (or any other build tool) should be able to consume the `reactNativeManifest` section and warn at build time if a configuration is invalid (either by having an invalid value or by having a mixture of invalid configurations).

It's outside the scope of this RFC to delve into details of which tool will implement which logic and we defer this to a future discussion.

### React Native Directory

The React Native direcotry can easily consume the `reactNativeManifest` by:
1. Query the NPM registry with [their API](https://github.com/npm/registry/blob/master/docs/REGISTRY-API.md) to obtain the repository information.
2. Query the repository information to obtain the `packge.json` and read the `reactNativeManifest` section.

## Future proof and extensibility

We believe that the `reactNativeManifest` section should be extensible as we believe new capabilities and features can be added there.
Specifically, we envision to evolve `reactNativeManifest` or `reactNativeManifest.capabilites` as follows:

1. Each addition of a new capability [requires a new RFC](https://github.com/react-native-community/discussions-and-proposals#proposals) to be approved.
2. Each RFC should define a new section in the `reactNativeManifest` with what's the intended use case.
3. Each RFC should define what are the sensible defaults and should refrain from breaking changes.
4. Each RFC should define the deprecation strategy for previous flags/features related to the same capability.

## Complete Example

Here we present a full example of how a `reactNativeManifest` section could look like for a library and an app.

For an app the section will look as follows:
    
```json
{
  "name": "my-awesome-app",
  "version": "1.2.3",
  "reactNativeManifest": {
    "version": "1.0",
    "type": "app",
    "capabilities": {
      "hermes": {
        "enabled": true
      },
      "newArch": {
        "enabled": true
      },
      "codegenConfig": {
        "name": "AppSpecs",
        "type": "all",
        "jsSrcsDir": ".",
        "android": {
          "javaPackageName": "com.facebook.fbreact.specs"
        }
      }
    }
  }
}
```

For a library the section will look as follows:

```json
{
  "name": "my-awesome-library",
  "version": "1.2.3",
  "reactNativeManifest": {
    "version": "1.0",
    "type": "library",
    "capabilities": {
      "hermes": {
        "enabled": true
      },
      "newArch": {
        "enabled": true
      },
      "codegenConfig": {
        "name": "LibrarySpecs",
        "type": "all",
        "jsSrcsDir": ".",
        "android": {
          "javaPackageName": "com.facebook.fbreact.specs"
        }
      }
    }
  }
}
```

### JSON Schema

The JSON Schema will be published as soon as we find an agreement on the key names and semantics.

## Alternatives

`reactNativeManifest` is, by design, fully original and new: there are other files that are used to define configurations for various aspect of react-native based projects (such as `react-native.config.js`, `app.json`, Expo's `app.config.js`, `expo-module.config.json`) but this one does **not** overlap with any of them. Explicitly, this one has the purpose of being filled only with capabilities and metadata to be read by package managers and build tools.

## Adoption strategy

If we decide to adopt this proposal, we'll have to broadcast this change to the ecosystem.
Ideally, we'll start by adding the `reactNativeManifest` section to the app template and to `create-react-native-library`.

We foresee a phase in which only a part of the userbase has this section configured and tool developers need to account for the absence of this section.

Once the v1.0.0 version of the shape is finalised, we can start integrating it across the board in various tools mentioned in [Proposed Tooling](#proposed-tooling) section. This first wave of support would have to be aligned around a given React Native release, say in 0.73 (which branch has not been cut).

After adding this first wave of support, and related documentation, we could more broadly communicate to the community and the maintainers how adding this section would benefit them too.

### Documentation update needed

If we agree on this proposal, we'll have to:

- Update the `reactnative.dev` website to mention this (and harmonize the current New Architecture and Hermes configuration pages)
- Update the app/library templates as mentioned
- Potentially add a CTA banner to React Native Directory and `align-deps` to introduce this field.

## Open questions

* `v1.0.0` of the schema needs to be finalized and agreed by stakeholders.
