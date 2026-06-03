---
title: DRAFT | Introducing `reactNativeManifest` to `package.json` for React Native specific metadata
author:
- Lorenzo Sciandra
- Nicola Corti
- Riccardo Cipolleschi
date: 25-01-2021
---

# RFC0588: [DRAFT] Introducing `reactNativeManifest` to `package.json` for React Native specific metadata

## Summary

> **Warning** ⚠️ This is the initial draft of this proposal. The goal of this draft is to collect feedback and help finalize the spec for this section.

This RFC introduces and formally discuss the concept of _React Native Manifest_, a set of metadata about a React Native project (either app or library). This metadata can be consumed by tooling to enhance the developer experience and support the React Native ecosystem.

This would be the entry point for all the official configurations supported by the core of React Native. Examples (not an exhaustive list) of these configurations could be:
* Whether the New Architecture is enabled or not.
* Which JS Engine should run in the app.
This will not be the place where we can add any kind of configuration.

Practically, this RFC proposes to add a `reactNativeManifest` section to the `package.json` file of a React Native app or library.
This new section will allow developers to follow a declarative approach to express capabilities of apps and libraries in a more formalized manner.

## Rationale

The need to specify **capabilities and metadata** for a react-native library is inherent to the React Native ecosystem. This need is so evident that various frameworks provided their own ways to specify those values. Look at `react-native.config.js` for the CLI or `app.json` for Expo.

Notice that we do not expect that frameworks stop using those files to configure **framework-specific** settings. What we would like to enforce with this proposal is to have single source of truth for what concern the React Native core settings.

Another issue is that, as of today, there is no simple programmatic way to know if a NPM package is a React Native library.
We have some heuristics, though:
* Presence of a `react-native:*` dependency in the `peerDependencies` section of the `package.json` ([example](https://github.com/RonRadtke/react-native-blob-util/blob/80628d418a5c81439dc2c1a1f05fae6406f0ba7f/package.json#L46C10-L49))
* Presence of React Native specific files such as `react-native.config.js` files
Those indicators are all non-exhaustive.

Similarly, there is no easy way to understand if a library is exposing a native module, a native component or not.
The lack of this information pushed the community to create workarounds to understand that. For Example, the React Native CLI tries to circumvent it by inspecting the native code and searching for subclasses of `*ReactPackage` using complex regular expressions ([example](https://github.com/react-native-community/cli/blob/605c542d18efcb02f217d3c85726fa73a79054c2/packages/cli-platform-android/src/config/findPackageClassName.ts#L35)) due to the lack of a declarative way to express this information.

Another practical example of this need is the [React Native Directory](https://reactnative.directory/) attempting to [programmatically infer](https://github.com/react-native-community/directory/pull/870) the New Architecture support from the presence of the `codegenConfig` fields in the `package.json` of a library. The `codegenConfig` field is not intended to carry semantic information about New Architecture support, and is definitely a non-exhaustive solution for this problem.

While we can continue to build tooling that tries to infer metadata and capabilities of a library, all these approach would never be precise and could provide misleading information.

We suggest to provide a declarative way to describe such capabilities.
This approach would yield several benefits:
* **predictability**: developers can easily understand what a library is capable of doing and if such capability matches their app.
* **tooling**: tooling can easily consume this information and provide a better experience to developers. For example, we could tailor `microsoft/rnx-kit`'s align-deps tool to consume this information to help users double-check whether all their dependencies are supporting the New Architecture. See [`align-deps` support](#align-deps) below
* **single source of thruth**: a single manifest will act as a source of truth for all those capabilities and flag definition, that can easily be consumed by different platforms to provide a unified way to enable/disable capabilities.
* **verifiability**: the manifest can be easily verified and linted by tooling to ensure that apps and libraries are not using an incompatible set of dependencies. Similarly, the JSON schema we publish can be used to validate the `reactNativeManifest` section.

## Use Cases

This section lists the use cases we envision for the `reactNativeManifest` section.

### Declaring the type of a project

For the sake of this RFC, we consider a React Native project as either an app or a library.
While an app project can contain native modules and components, it's **not** possible for an app to "consume" another app as a library. This would break the native build system invariants (i.e. a single `AndroidManifest.xml` and `Info.plist` file per app, etc.). If an app projects wishes to distribute some of its native modules or components, they need to be extracted into a library project.

We're proposing to declare if a project is either a library or an app by adding the `type` field (as `"app"` or `"library"`) to the `reactNativeManifest` section of the `package.json` file.

```json
{
  "reactNativeManifest": {
    "type": "library"
  }
}
```

This is a mandatory field that needs to be specified in all the apps and libraries.

### New and Old Architecture support

One of the primary driver of this proposal is the [New React Native Architecture support](https://reactnative.dev/docs/the-new-architecture/landing-page). Specifically we believe that the ecosystem is currently affected by:
* Lack of a declarative way to define if a library is compatible with the New Architecture
  * As described in the [rationale](#rationale), tooling needs to infer this information from the presence of the `codegenConfig` field in the `package.json` of a library. Libraries might decide not to use Codegen and still be compatible with the New Architecture, so this method is not exhaustive.
  * The alternative at this stage is to inspect the code and check whether the API callsites are New Architecture or Old Architecture compatible.
* Lack of a declarative way to enable the New Architecture on both platforms
  * Currently the New Architecture is enabled by a Gradle Property `newArchitectureEnabled` on Android in the `gradle.properties` file and by invoking `RCT_NEW_ARCH_ENABLED=1 bundle exec pod install` on iOS.
  * As of today, there is no way to enable the New Architecture for an app project for both platforms
  * Moreover, the difference in how the New Architecture is enabled leads to a scenario where you can't statically know if an app supports New Architecture or not (as the New Architecture support is known at  `pod install` time and is not codified in the codebase)

Therefore we propose to add the `newArchitecture` section to the `reactNativeManifest.capabilities` of **both apps and libraries** with the following semantic:
* For Apps: `newArchitecture.enabled==true` means that the app wants to use the New Architecture.
* For Libraries: `newArchitecture.enabled==true` means that the library is compatible with the New Architecture.

Tools can be built on top of this information to check that an app with `newArchitecture.enabled==true` is not accepting libraries with `newArchitecture.enabled==false` and to warn against library that don't have the key specified.

The setup would look as follows for both apps and libraries:

```json
{
  "reactNativeManifest": {
    "capabilities": {
      "newArchitecture": {
        "enabled": true
      }
    }
  }
}
```

This section will allow also for split configuration between platforms:

```json
{
  "reactNativeManifest": {
    "android": {
      "capabilities": {
        "newArchitecture": {
          "enabled": true
        }
      }
    },
    "ios": {
      "capabilities": {
        "newArchitecture": {
          "enabled": true
        }
      }
    }
  }
}
```

If omitted, the `newArchitecture.enabled` would have a value of `false` (this may change in the future).

Notice that **libraries** might be supporting both architecture. So, the `capabilities` field can also support an `oldArchitecture` field of the same shape of the `newArchitecture`. The following example shows how to use both properties.

```json
"reactNativeManifest": {
  "capabilities": {
    "newArchitecture": {
      "enabled": true
    },
    "oldArchitecture": {
      "enabled": true
    }
  }
}
```

If omitted, the `oldArchitecture.enabled` would have a value of `true` (this may change in the future), to simplify the adoption of the manifest. In fact, the vast majority of React Native libraries are now supporting the Old Architecture and only a subset support both of them.

The `oldArchitecture` property is **ignored** by apps: this is needed to avoid ambiguous setup where an app declares both architectures as enabled.
In other words, the `newArchitecture` property has always precedence in an app.

The precedence rules are that platform-specific settings override general settings. So, for example:

```json
{
  "reactNativeManifest": {
    "capabilities": {
      "newArchitecture": {
        "enabled": true
      }
    },
    "android": {
      "capabilities": {
        "newArchitecture": {
          "enabled": false
        }
      }
    },
  }
}
```

would means that, in general, the app/library support the New Architecture, but not for Android.

##### Planning forward: what happen when the New Architecture will be the default?

The manifest is versioned, and we will use the versioning system to track breaking changes and different semantics.

For version `1.0.0` the semantic would be:
* `newArchitecture` default value set to `false`

In a future version, let's say `K.0.0`, the semantic would be:
* `newArchitecture` default value set to `true`
* `newArchitecture` field deprecated

In a more future version, let's say `N.0.0` (where N > K), the semantic would be:
* `newArchitecture` field removed, together with the Old Architecture code.

Notice that this system can also help us to track unmaintained/old packages: they would either not have the manifest at all or have old version of the manifest with its version's semantic.

#### `codegenConfig` support

Similarly to the New Architecture support metadata, the `codegenConfig` is a key metadata of the New Architecture build pipeline.
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

#### Hermes Support

Similarly to the New Architecture support, the current way to enable/disable the Hermes engine is toggled by using a Gradle Property `hermesEnabled` on Android and changing the `:hermes_enabled` property in the Podfile.

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

If omitted, the `hermes.enabled` would have a value of `true`.

The precedence rules are that platform-specific settings override general settings. So, for example:

```json
{
  "reactNativeManifest": {
    "capabilities": {
      "hermes": {
        "enabled": true
      }
    },
    "android": {
      "capabilities": {
        "hermes": {
          "enabled": false
        }
      }
    },
  }
}
```

would means that the app will run with Hermes for all the platforms it supports but not for Android.

#### React Native Release feature flagging

We currently have a number of different files where capabilities of React Native can be toggled to enable/disable features in the runtime.
For example we have:
* [android/gradle.properties](https://github.com/facebook/react-native/blob/main/packages/react-native/template/android/gradle.properties) to specify key-value build time properties for Android
* [android/app/build.gradle](https://github.com/facebook/react-native/blob/main/packages/react-native/template/android/app/build.gradle) to specify build time configuration for Android
* [ReactNativeFeatureFlags.js](https://github.com/facebook/react-native/blob/main/packages/react-native/Libraries/ReactNative/ReactNativeFeatureFlags.js) which contains runtime feature toggles for the Javascript layer.
* [ReactFeatureFlags.java](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/config/ReactFeatureFlags.java) which contains runtime feature toggles for Android.
* [ReactNativeConfig.cpp](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactCommon/react/config/ReactNativeConfig.cpp) which contains runtime feature toggles for the C++ layer.

Most of those config flags are undocumented and they're not verified for invalid configuration. Specifically, some feature flags require to be toggled on multiple layers, and it's easy to forget to toggle one of them. Invalid combinations of those flags could lead to unexpected runtime behavior.

We propose to define `reactNativeManifest` as the **preferred entry point** for all the user facing entry points. All the tooling should be adapted to consume the information from the `reactNativeManifest` section and have sensible defaults defined by the tooling itself.

It's outside the scope of this RFC to declare all the new sections we intend to add, and we defer to the [Future proof and extensibility](#future-proof-and-extensibility) section for the process on how to include a new key to the `reactNativeManifest` section.

#### Out-of-Tree Platforms

Although this proposal does not cover directly out-of-tree platforms, we strongly suggests maintainers of out-of-tree platforms to follow the same path.

Tools written on top of the React Native Manifest would be able to work with iOS, Android and all the out of tree platforms that they support. They should follow the same precedence rules: if there is no platform specified, then the main flag would take precedence. If there is a platform specified, then that platform setup would have precedence.

So, for example, given the manifest:

```json
{
  "reactNativeManifest": {
    "windows": {
      "capabilities": {
        "hermes": {
          "enabled": false
        }
      }
    },
  }
}
```

It means that for all the platforms but Windows, React Native will use Hermes as JS engine (`hermes.enabled` default value is `true`). For Windows specifically, it would be disabled.

As an extra note, please be careful to **follow the same semantics** as the React Native Core: in this way we can avoid confusion for our users.

## Excluded use cases

This section lists the use cases where we believe the `reactNativeManifest` section should **not** be used.

### React Native version support

For the time being, we're not planning to add a `version` section to the `reactNativeManifest` section for libraries. The rationale is that we believe that the `react-native` version should be specified in either the `peerDependencies` or the `engines` section of the `package.json` of the project. This follows the idiomatic way to specify the version of a dependency in a NodeJS project.

### TurboModule/Fabric toggles

At the time of writing, we prefer not to offer a dedicated section to toggle Fabric/TurboModule capability inside `reactNativeManifest`. The rationale is that we believe that the `newArchitecture` section is sufficient to expose the New Architecture to users.
Selectively toggling Fabric/TurboModule is a more advanced feature. We believe we'll still be offering a more advanced way to enable or disable those pieces of the New Architecture infrastructure, but at the current state, our preference is not to expose such capability in the top level `reactNativeManifest` section.

## Proposed Tooling

This section lists the tooling we believe could be built on top of `reactNativeManifest` to improve the developer experience.

### Build Logic support

Build tools such as Gradle/CocoaPods or others should account for the `reactNativeManifest` section and use it to configure the build pipeline accordingly to this and future RFCs. Specifically they should follow this logic for a generic capability `foo`.

1. If the capability `foo` is specificed in the `reactNativeManifest.capabilities` section, use the value specified in the `reactNativeManifest.capabilities.foo` section.
   1. If the user is **also** specifying the capability `foo` in the build tool specific configuration (e.g. inside `gradle.properties` for Android) behave as follows:
      1. If the two values are **compatible**, use them without notifying the user.
      1. If the two values are **incompatible**, notify the user and use the value specified in the `reactNativeManifest.capabilities.foo` section will prevail and the value specified in the build tool specific configuration will be ignored.
1. If the capability `foo` is **not** specificed in the `reactNativeManifest.capabilities` section, honor the value specified in the build tool specific configuration or the default value if not specified.

### <a name="align-deps"></a>`align-deps` support

This RFC originated from a [conversation between Meta and Microsoft](https://github.com/microsoft/rnx-kit/issues/1863) to use `align-deps` as key feature in the New Architecture rollout support, in helping developers understand if a library they're using is compatible with the New Architecture or not. Very quickly it became clear that there is currently no straightforward way to know if a library indeed has that support.

Therefore we believe `align-deps` can benefit from this information and can be extended to offer New Architecture support information.

### React Native CLI support

Ideally the CLI (or any other build tool) should be able to consume the `reactNativeManifest` section and warn at build time if a configuration is invalid (either by having an invalid value or by having a mixture of invalid configurations).
For example, the CLI could use [jsonschema](https://npmjs.com/package/jsonschema) at build time and validate the `package.json` against the schema we provide.

It's outside the scope of this RFC to delve into details of which tool will implement which logic and we defer this to a future discussion.

### React Native Directory

The React Native directory can easily consume the `reactNativeManifest` by:
1. Query the NPM registry with [their API](https://github.com/npm/registry/blob/master/docs/REGISTRY-API.md) to obtain the repository information.
2. Query the repository information to obtain the `packge.json` and read the `reactNativeManifest` section. By putting these configurations in the package.json, React Native Directory can fetch only this file, rather than downloading the whole package and crawling its files.

## Future proof and extensibility

We're versioning the schema using the `schema` key in the `reactNativeManifest` section. This allows us to evolve the schema in the future without breaking changes. We believe that the `schema` key should be a semver version string.

We believe that the `reactNativeManifest` section should be extensible as we believe new capabilities and features can be added there.
Specifically, we envision to evolve `reactNativeManifest` or `reactNativeManifest.capabilites` as follows:

1. Each addition of a new capability [requires a new RFC](https://github.com/react-native-community/discussions-and-proposals#proposals) to be approved.
2. Each RFC should define a new section in the `reactNativeManifest` with what's the intended use case.
3. Each RFC should define what are the sensible defaults and should refrain from breaking changes.
4. Each RFC should define the deprecation strategy for previous flags/features related to the same capability, if needed.

## Complete Example

Here we present a full example of how a `reactNativeManifest` section could look like for a library and an app.

For an app the section will look as follows:

```json
{
  "name": "my-awesome-app",
  "version": "1.2.3",
  "reactNativeManifest": {
    "schema": "1.0.0",
    "type": "app",
    "capabilities": {
      "hermes": {
        "enabled": true
      },
      "newArchitecture": {
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
    "schema": "1.0.0",
    "type": "library",
    "capabilities": {
      "newArchitecture": {
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

We foresee a phase in which only a part of the user base has this section configured, and tool and developers need to account for the absence of this section.

Once the v1.0.0 version of the shape is finalised, we can start integrating it across the board in various tools mentioned in [Proposed Tooling](#proposed-tooling) section. This first wave of support would have to be aligned around a given React Native release, say in 0.74 (which branch has not been cut).

After adding this first wave of support, and related documentation, we could more broadly communicate to the community and the maintainers how adding this section would benefit them too.

Furthermore, we will provide validation scripts to verify that the new field do respect the schema described by its version.

### Documentation update needed

If we agree on this proposal, we'll have to:

- Update the `reactnative.dev` website to mention this (and harmonize the current New Architecture and Hermes configuration pages)
- Update the app/library templates as mentioned
- Potentially add a CTA banner to React Native Directory and `align-deps` to introduce this field.

## Open questions

* `v1.0.0` of the schema needs to be finalized and agreed by stakeholders.
