---
title: React Native Test App as a Package
author:
  - Tommy Nguyen
date: February 2020
---

# RFC0000: React Native Test App as a Package

## Summary

Provide a React Native test app for all platforms as a package.

A working prototype of this can be found here:
https://github.com/microsoft/react-native-test-app

## Motivation

Maintaining and keeping React Native up to date with the latest version can be
time consuming. Things can break if you're not careful, even with
[React Native upgrade helper](https://react-native-community.github.io/upgrade-helper/).
Not all developers have much experience in native development on a single
platform, let alone _all_ platforms they want to support. Adding support for
another platform, such as Windows or macOS, can be overwhelming.

Consider the following scenarios:

- As maintainers of a community package in our spare time, we don't have the
  capacity to create and maintain test apps.

  - Our inner development loop consists of using `npx react-native init`, and
    testing things manually.
  - Alternatively, we have set up CI for iOS and Android, and would love to add
    support for Windows and macOS but lack the resources.

- We have a monorepo with multiple separate experiences that all need test apps.
  It is a pain having to upgrade all of the test apps individually with every
  release. Our upgrade process can sometimes take weeks to perform.

- As a consumer of React Native, we should try out release candidates to find
  issues early and contribute back, but due to one of the scenarios above, there
  is simply too much friction.

The proposed package will, at minimum, provide a customizable test app for all
the major platforms that React Native currently runs on, i.e. iOS, Android,
Windows, and macOS. The ultimate goal of this package is to make adding support
for a platform as simple as `npm add --save-dev`, and upgrading to latest React
Native is just a version bump.

We expect the package will:

- Help developers set up a consistent native environment for both local
  development and CI agents
- Reduce or eliminate the friction to upgrade React Native, and try out release
  candidates
- Reduce or eliminate the overhead of adding support for new platforms

With common test apps, we will also be able to provide additional value for
everyone simultanously. For instance:

- We will handle upgrading of the test apps to support the latest mobile SDK
- We can provide a common infrastructure for native testing
- We can provide optional modules for commonly used services, such as auth or
  App Center integration

## Detailed design

Given an existing React Native app/library, we imagine the ideal flow for
**initial integration** with the native test apps should be:

1. Create folder/package for the test app to live in.
   - This package should have a dependency on the app/library to be tested, and
     on the test app package.
2. Create a manifest to declare entry points. Optionally, it should also be
   possible to declare initial props, native source files and resources.
   - This is a **one-time cost**, and should be stable across versions.
   - If there is a single entry point, the app may boot directly into this app.
   - Otherwise, if there are multiple entry points, we will show them in a list.
     - It should still be possible to have the app boot into one of these via
       deep linking.
   - Native dependencies should be taken care of by auto-linking behind the
     scenes.
3. Run `npm install` inside this folder as one would usually do.
4. Depending on the desired target platforms, the next steps are:
   - Android:
     - Make minimal changes to `build.gradle` (one-time cost)
     - Open the project in Android Studio
   - iOS:
     - Make minimal changes to `Podfile` (one-time cost)
     - Run `pod install`
     - Open the newly generated Xcode workspace
   - Windows:
     - <@afoxman or @acoates-ms to fill in>
   - macOS:
     - See steps for iOS

A hypothetical package should end up looking something like below:

```
my-awesome-component
├── example
│   ├── App.js
│   ├── build.gradle (new/modified)
│   ├── package.json
│   ├── Podfile (new/modified)
│   └── yarn.lock
├── my-awesome-component.podspec
├── my-awesome-component-dev.podspec (new, optional)
├── package.json
├── src
└── yarn.lock
```

For a hypothetical monorepo:

```
my-monorepo
├── package.json
├── packages
│   ├── my-awesome-component-android
│   │   ├── build.gradle (new/modified)
│   │   ├── package.json
│   │   └── src
│   ├── my-awesome-component-core
│   │   ├── package.json
│   │   └── ...
│   └── my-awesome-component-ios
│       ├── my-awesome-component.podspec
│       ├── my-awesome-component-dev.podspec (new, optional)
│       ├── package.json
│       ├── Podfile (new/modified)
│       └── src
└── yarn.lock
```

Platform specific project files should no longer be checked in. They are either
provided by React Native Test App package or are generated at build time.

Once the test app package is configured, **adding an additional platform**
should only require doing one of the platform specific substeps of step 4.

**Upgrading React Native** to the latest version, or any desired version, should
only require bumping the version of this package.

This package should not require additional tools other than the ones required by
React Native itself, and should not fundamentally change how a package is
structured.

### The Manifest

The manifest will be the file where we declare all our entry points, and
resources that should be included.

We haven't decided what this manifest looks like. Our initial thought is to use
React Native CLI's
[configuration format](https://github.com/react-native-community/cli/blob/master/docs/configuration.md)
as a starting point. This is what
[auto-linking](https://github.com/react-native-community/cli/blob/master/docs/autolinking.md#how-does-it-work)
uses today, and we can extend it to support our scenarios.

For iOS (and macOS), we will need to extend the format to include additional
native source files and assets. An Xcode project generated today, will include a
main app target, and two test targets: One for "normal" tests, and one for UI
tests. We will need to be able to generate `.podspec` files that can support
all.

Entry points will also be declared in the manifest. Entry points tell the test
app how to launch your app. For instance, given the declaration below:

```js
module.exports = {
  ...
  "assets": [
    ...
  ],
  "entryPoints": [
    {
      "name": "MyFeature",
      "entryPoint": "MyFeatureTestScreen",
      "initialProps": {
        "showStatus": true
      }
    },
    {
      "name": "MySecondFeature",
      "entryPoint": "MySecondFeatureTestScreen",
      "presentationStyle": "popover"
    }
  ]
}
```

The mobile test app could look like this:

```
┌──────────────────────┐
│                      │
│RN Test App           │
├──────────────────────┤
│ MyFeature           >│
│ ─────────────────────┤
│ MySecondFeature     >│
├──────────────────────┤
│                      │
│                      │
│                      │
│                      │
│╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱╲│
```

Tapping on one of these cells will launch the corresponding app. If defined,
`initialProps` will be passed to the appropriate method call, and
`presentationStyle` will tell the test app how to present the new view.

One could also forgo the name, in which case the `entryPoint` string will be
used instead. If only one entry point is declared, the test app should launch
directly into that app.

Alternatively, we should be able to omit the `entryPoints` block and retrieve
the list of entry points from
[AppRegistry](https://github.com/facebook/react-native/blob/master/Libraries/ReactNative/AppRegistry.js#L165)
instead.

All of this should be transparent to the consumer; they should only have to deal
with the manifest.

## Drawbacks

- Package maintainers no longer have full control of the test app
- Bugs in the test app may take longer to get fixed
- Limitations in existing tools can cause confusions, e.g. files are not
  added/removed until `pod install`/`update` is run

## Alternatives

### Electrode

[Electrode Native](https://www.electrode.io/) is a platform by Walmart Labs
that provides an opinionated, batteries-included solution for integrating React
Native components into existing apps. It includes everything you'll ever need
such as React Native initialization code, code generators, code push, facilities
to test individual components etc.

From a cursory glance, it seems that Electrode decides which React Native
version will be embedded, and which bundler to use. Whereas one of the main
goals of React Native Test App is making it simpler to switch between React
Native versions, and it doesn't care whether Metro or Haul is used. Because
Electrode is all-inclusive, it seems to be overkill for single component
packages. You just want a test app for development and run tests on CI.

### Expo

[Expo](https://expo.io/) provides an app called Expo client that includes
multiple versions of the runtime in a single binary and makes it easy to switch
between them. Each version of the runtime uses a single version React Native.

[Snack](https://snack.expo.io/) is an in-browser editor like
[CodeSandbox](https://codesandbox.io/) for React Native apps. You can choose the
version of Expo SDK to target using a drop-down menu. A specific version of the
SDK usually maps to a distinct React Native version, e.g. SDK 36 uses React
Native 0.61 and SDK 35 uses React Native 0.59.

This is the quickest and easiest way to test React Native libraries for iOS,
Android, and web, provided that they depend on primitives in React Native core
or the most common dependencies in the ecosystem like
`react-native-gesture-handler`, `react-native-reanimated`, `react-native-maps`,
`react-native-svg`, and so on.

There are two reasons why this isn't feasible for the particular use case
described in this proposal. First, if a library includes its own native
dependencies, it cannot be run inside of Expo client unless the appropriate
native code is also included in the client at build-time. Second, Expo client
apps are not yet built for Windows and macOS. You would not be able to test on
those platforms.

### React Native CLI

[This tool](https://github.com/react-native-community/cli) creates a new project
with a specifiable React Native version. It can be further customized with
templates. We can see this test app package being part of the default template,
with a single entry point defined.

### RNTester

[RNTester](https://github.com/facebook/react-native/tree/master/RNTester) is an
app that showcases React Native views and modules. It is currently tightly
coupled to React Native and requires building the framework with the app.

## Adoption strategy

- Get buy-in from the community; we want to make sure that this package aligns
  with what package maintainers expect of a test app
- Submit the first few PRs to volunteering react-native-community packages to
  show how it can be integrated

## How we teach this

- Point to real PRs showing how it can be integrated
- Make it simple to get started with, e.g.:
  - A library template for
    [react-native-cli](https://github.com/react-native-community/cli) so
    developers can run
    `react-native init ProjectName --template "react-native-library-template"`
    to generate a project with test apps for all platforms ready to go
  - Ship as default with library project generators such as
    [Bob](https://github.com/react-native-community/bob) or
    [create-react-native-module](https://github.com/brodybits/create-react-native-module)

## Unresolved questions

- Manifest format
- Versioning scheme

## Related discussions

- https://github.com/react-native-community/discussions-and-proposals/issues/96
- https://github.com/facebook/react-native/issues/6292
