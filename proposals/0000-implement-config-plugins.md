---
title: Support Expo Config Plugins in React Native apps
author:
- Tomasz Misiukiewicz
- Michał Pierzchała
date: 20-04-2023
---

# RFC0000: Support Expo Config Plugins in React Native apps

## Summary

Upstream part of Expo Config Plugins into React Native core to support modifying application's config without having to manually edit the native files.

## Basic example

https://user-images.githubusercontent.com/13985840/229797752-81009e28-3153-4702-b52f-14602789a87b.mov

In this example, Android permissions defined in `app.json` are added to `AndroidManifest.xml` file, and a new key is added into `Info.plist` file based on a custom plugin:

```js
module.exports = function (config, { apiKey }) {
    // Ensure the objects exist
    if (!config.ios) {
      config.ios = {};
    }
    if (!config.ios.infoPlist) {
      config.ios.infoPlist = {};
    }
  
    // Append the apiKey
    config.ios.infoPlist['MY_CUSTOM_NATIVE_IOS_API_KEY'] = apiKey;
  
    return config;
  };
```

### Code samples

[Example implementation of `prebuild` command into React Native CLI, based on Expo Config Plugins](https://github.com/react-native-community/cli/compare/main...TMisiukiewicz:cli:prebuild)

This is a quick PoC on how we could possibly adapt config plugins into CLI. Prebuild command is very similar to the one from Expo CLI, it just doesn't contain Expo-related things

[Possible implementation in React Native app, and how it affects native files](https://github.com/TMisiukiewicz/MyTestApp/compare/main...test-appjson)

This PoC shows how `prebuild` command affects the native code. Find `app.json` file first, see how config for both platforms is defined here, and then go through all the changes in the native files to check what changes were made after using `prebuild`

[Create custom plugin for `react-native-image-picker` library](https://github.com/react-native-image-picker/react-native-image-picker/compare/main...TMisiukiewicz:react-native-image-picker:create-plugin)

[Usage of created custom plugin in the example app](https://github.com/TMisiukiewicz/MyTestApp/compare/expo-custom-mod...add-rn-lib)

To prove that it would benefit libraries maintainers and community, I created a simple plugin for `react-native-image-picker` library, which allows to set the description messages for required permissions via `app.json`. Plugin is applied in `plugins` array and it's overriding `Info.plist` values.

## Motivation
According to the [Expo docs](https://docs.expo.dev/guides/config-plugins/), config plugins are a system for extending Expo config and customizing the prebuild phase of managed builds. Expo CLI is using it internally to generate and configure all the native code for managed projects.

Based on that, for the purpose of this RFC, I'd define "Config Plugins" as a **set of configuration options for React Native app, providing a standardized way of generating and configuring native code for any platform**.

The main benefit would be to **make the process of upgrading apps to the latest React Native version much simplier than it is now**. Very often it requires some additional manual steps on the native side, resolving conflicts etc. With Expo Config Plugins implemented into React Native, it would be possible to store all the native-related config on the JS side and apply it into native files whenever it's needed, e.g. while running `react-native upgrade`.

Currently using config plugins is available only to projects based on Expo. As showcased in the code samples above, it’s possible to just use `@expo/config-plugins` today, however it poses a risk of incompatibilities for built-in plugins that rely on specific Expo setup that may not be there in every RN app.

Another motivation to move it into core is to tie up Config Plugins to the React Native app template. Currently, these two components are separate, and despite our efforts to establish their compatibility through conceptual work, further modifications to the template could potentially result in disruptions. **By upstreaming it into the core, we can create a seamless connection between Config Plugins and the RN template, mitigating the risk of potential breakages caused by future changes**.

We believe Expo provides superior DX for most of the app teams out there. Due to historical reasons, there are myths around what Expo can and can’t do, that are vastly invalid. We hope this proposal, bridging Expo ideas into RN CLI apps, will help bust those myths, push more users to Expo as a result and improve the experience of those that for any reasons don’t won’t to do so.

## Detailed design

### Proposal of the new workflow
![image](https://user-images.githubusercontent.com/13985840/233337419-3439dd5d-ce7e-4298-a021-4bf49f184be3.png)
This diagram shows how the new workflow would work after doing all the required changes. "Config plugins" would become a part of React Native core, which then can be consumed by CLI, Expo, libraries and others.

> This implementation might be also related to [RFC: introduce reactNativeMetadata to package.json](https://github.com/react-native-community/discussions-and-proposals/pull/588/files). At this moment design is covering usage of `app.json` file, but additionally it might fit into linked RFC as well.

The very first step would be upstreaming non-Expo related code of `@expo/config-plugins` and `@expo/config-types` into React Native core as a separate package in monorepo. It contains all logic and helpers needed to modify native side of RN project easily from JS side. We'd handle discussions with Expo to coordinate what parts of config plugins can be safely upstreamed into core. It has to be done in a way where Expo would still able to extend it on their side with all the stuff related to Expo, like EAS Builds etc.

Upstreaming it will unlock a few new paths that React Native could follow. First of all, and probably most important, **it can change the way of generating native code**. It would become possible to add platform-specific folders like `android` and `ios` to `.gitignore` by default and keep them out of the repository unless it's needed (e.g. when it's not possible to do some native-side changes with using config plugins). These folders would be automatically (re)generated when running one of the following commands:
- `run-ios`
- `run-android`
- `build-ios`
- `build-android`
- `upgrade`

This would require having `@react-native/config-plugins` as a dependency of CLI.

> If it's possible in the future, we could move platform-specific code from CLI into the core to avoid that. It might be related to the [Move iOS and Android Specific Code to Their Own Packages](https://github.com/react-native-community/discussions-and-proposals/pull/49/files) RFC.

All of the commands would use newly created method that would apply all the changes defined in `app.json` file. Expo is using this file to determine how to load the app config in Expo Go and Expo Prebuild. With config plugins implemented, this file could work the same way as in Expo, but, by default, it would not support Expo-related properties (Expo will still be able to easily extend this config to match their needs). You can get more info about configuration with app.json in Expo [here](https://docs.expo.dev/workflow/configuration). **It will also become an opportunity to create configurations for Out-of-Tree Platforms.**

Here's the list of Expo Config properties, divided by those that React Native could possibly handle out of the box, and the Expo-specific ones: 

#### Platform-independent
##### Possible to adopt by RN
- `name` - The name of the app.
- `platforms` - Platforms that project supports.
- `orientation` - Locks the app to a specific orientation with portrait or landscape. Defaults to no lock. Valid values: `default`, `portrait`, `landscape`
- `scheme` - URL scheme to link into the app.
- `primaryColor` - determines the color of the app in the multitasker.
- `icon` - local path or remote url to use for app's icon
- `locales` - overrides by locale for System Dialog prompts like permission boxes
- `plugins` - config plugins for adding extra functionality to the project
##### Expo-specific
- `slug` - friendly URL name for publishing
- `owner` - name of the Expo account that owns the project
- `currentFullName` - auto generated Expo account name and slug used for display purposes
- `originalFullName` - auto generated Expo account name and slug used for services like Notifications and AuthSession proxy
- `privacy` - project page access restrictions
- `sdkVersion`- Expo sdkVersion to run the project on.
- `splash` - Configuration for loading and splash screen.
- `githubUrl` - linking GH project with Expo project page
- `userInterfaceStyle` - forces app to always use the light/dark/automatic mode. Requires `expo-system-ui` to be installed
- `backgroundColor` - background color for the app, behind all React views. Requires `expo-system-ui` to be installed
- `androidStatusBar` - configuration of status bar on Android, requires `expo-status-bar`
- `androidNavigationBar` - config for bottom navigator bar. Requires `expo-navigation-bar`
- `developmentClient` - settings specific to running app in dev client
- `extra` - extra fields to pass, accessible via `Expo.Constants.manifest.extra`
- `updates` - configuration for how and when the app should request OTA updates
- `isDetached` - is app detached
- `detach` - extra fields needed by detached apps
- `assetBundlePatterns` - array of file glob strings which point to assets bundled within app binary
- `notification` - configure for remote push notifications
- `jsEngine` - specifies the JS engine for apps, supported only on EAS Build
- `hooks` - configuration for scripts to run to hook into the publish process
- `experiments` - experimental features related to Expo SDK

#### iOS
##### Possible to adopt by RN
- `bundleIdentifier` - iOS bundle identifier notation unique name for the app.
- `buildNumber` - Build number for your iOS standalone app. Corresponds to `CFBundleVersion`
 and must match Apple's [specified format](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion).
- `icon` - Local path or remote URL to an image to use for app's icon on iOS. If specified, this overrides the top-level `icon` key. Use a 1024x1024 icon which follows Apple's interface guidelines for icons, including color profile and transparency.
- `bitcode` - Enable iOS Bitcode optimizations in the native build. Accepts the name of an iOS build configuration to enable for a single configuration and disable for all others, e.g. Debug, Release.
- `supportsTablet` - Whether standalone iOS app supports tablet screen sizes. Defaults to `false`
- `isTabletOnly` - If true, indicates that iOS app does not support handsets, and only supports tablets.
- `userInterfaceStyle` - Configuration to force the app to always use the light or dark user-interface appearance, such as "dark mode", or make it automatically adapt to the system preferences. If not provided, defaults to `light`
- `infoPlist` - Dictionary of arbitrary configuration to add to app's native Info.plist.
- `entitlements` - Dictionary of arbitrary configuration to add to app's native *.entitlements (plist).
- `associatedDomains` - An array that contains Associated Domains for the app.
- `accessesContactNotes` - A Boolean value that indicates whether the app may access the notes stored in contacts. [Receiving a permission from Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_contacts_notes) is required before submitting an app for review with this capability.
- `runtimeVersion` - The runtime version associated with this manifest for the iOS platform. If provided, this will override the top level runtimeVersion key.
- `googleServicesFile` - location of GoogleService-Info.plist` file for configuring Firebase
- `requireFullScreen` - indicates that iOS app does not support Slide Over and Split View on iPad

##### Expo-specific
- `publishManifestPath` - manifest for the iOS version written to this path during publish
- `publishBundlePath` - bundle for the iOS version written to this path during publish
- `splash` - Configuration for loading and splash screen for iOS apps.
- `backgroundColor` -  background color for the app, behind all React views. Requires `expo-system-ui` to be installed
- `appStoreUrl` - link to your store page from your Expo project page if app is public
- `usesIcloudStorage` - indicating if the app uses iCloud Storage for Expo's DocumentPicker
- `usesAppleSignIn` - indicating if the app uses Apple Sign-In with Expo's Apple Authentication
- `jsEngine` - Specifies the JavaScript engine for iOS apps. Supported only on EAS Build
- `config` - config for e.g. Google Maps API, Mobile Ads API etc

#### Android
##### Possible to adopt by RN
- `package` The package name for Android app. It needs to be unique on the Play Store
- `versionCode` Version number required by Google Play. Must be a positive integer.
- `backgroundColor` The background color for Android app, behind any of React views. Overrides the top-level `backgroundColor` key if it is present.
- `icon` Local path or remote URL to an image to use for app's icon on Android. If specified, this overrides the top-level `icon` key. 1024x1024 png file is recommended (transparency is recommended for the Google Play Store). This icon will appear on the home screen.
- `adaptiveIcon` Settings for an Adaptive Launcher Icon on Android
- `permissions` List of permissions used by the app.
- `blockedPermissions` List of permissions to block in the final `AndroidManifest.xml`.
- `intentFilters` Configuration for setting an array of custom intent filters in Android manifest.
- `allowBackup` Allows user's app data to be automatically backed up to their Google Drive. If this is set to false, no backup or restore of the application will ever be performed (this is useful if your app deals with sensitive information). Defaults to the Android default, which is `true`
- `softwareKeyboardLayoutMode` Determines how the software keyboard will impact the layout of the application. This maps to the `android:windowSoftInputMode` property. Defaults to `resize`. Valid values: `resize`, `pan`
- `runtimeVersion` The runtime version associated with this manifest for the Android platform. If provided, this will override the top level runtimeVersion key.
- `googleServicesFile` - Location of the `GoogleService-Info.plist` file for configuring Firebase. Including this key automatically enables FCM in the app.

##### Expo-specific
- `publishManifestPath` - manifest for the iOS version written to this path during publish
- `publishBundlePath` - bundle for the iOS version written to this path during publish
- `splash` Configuration for loading and splash screen for Android apps.
- `userInterfaceStyle` - forces app to always use the light/dark/automatic mode. Requires `expo-system-ui` to be installed
- `playStoreUrl` - URL to your app on the Google Play Store, used to link Expo project with store page
- `jsEngine` - Specifies the JavaScript engine for iOS apps. Supported only on EAS Build
- `config` - config for e.g. Google Maps API, Mobile Ads API etc

The CLI would look for any changes in the `app.json` file and regenerate the platform-specific folders when running one of the React Native commands.

Additionally, **it would become possible to create custom plugins within the app. Since the plugins would be available to import directly from the core, it would not need any additional setup on the developer side**. E.g. developer can easily create a custom plugin:

`./plugins/withStringsXml.js`:
```js
const { AndroidConfig, withStringsXml } = require('@react-native/config-plugins')

module.exports = function (config) {
    return withStringsXml(config, (conf) => {
      conf.modResults = AndroidConfig.Strings.setStringItem(
        [
          {
            _: 'true',
            $: {
              name: 'test_value',
              translatable: 'false'
            }
          }
        ],
        conf.modResults
      )
      return conf
    })
}
```
`app.json`:
```diff
{
  "name": "MyTestApp",
  "displayName": "MyTestApp",
+ "plugins": ["./plugins/withNewString.js"]
}
```
In the example above, custom plugin is using `withStringsXml` mod. Mods are async functions that modify native files. A list of the mods available for custom plugins:

#### Android
- `withAndroidManifest`
- `withStringsXml`
- `withAndroidColors`
- `withAndroidColorsNight`
- `withAndroidStyles`
- `withMainActivity`
- `withMainApplication`
- `withProjectBuildGradle`
- `withAppBuildGradle`
- `withSettingsGradle`
- `withGradleProperties`

#### iOS
- `withAppDelegate`
- `withInfoPlist`
- `withEntitlementsPlist`
- `withExpoPlist`
- `withXcodeProject`
- `withPodfileProperties`

It would also be possible to use all of the helpers currently available in `@expo/config-plugins`, like `AndroidConfig.Strings.setStringItem` in the example above.

The mods can be also are appended to the mods object of the configuration. This section varies from the other parts of the configuration as it is not serialized after the initial reading. It's possible to utilize it to execute operations while generating code. In general, creating custom plugins with mod functions are simpler to work with, so it's recommended to use it instead of using `mods` object in `app.json`.

### `prebuild` command
**Additionally, React Native CLI would have a couple of new commands, similar to `prebuild` known from Expo CLI**. The method applying config plugins into native code would run with the commands mentioned above, but, `prebuild` command would additionally add the native files into repository to make sure developers can safely make any manual changes in native folders. If there is a need of prebuilding only one of the platforms, it would be possible to use `prebuild-android` and `prebuild-ios` commands. Prebuild would run all config plugins to apply all of the necessary changes. There should also be a possibility of using `--clean` flag with prebuild to install fresh copy of native directories and apply all config plugins. It should be used only if there are no manual changes made to native files.

### Library development

These changes would possibly also affect libraries development and allow maintainers to use this API within libraries. Based on the [Expo tutorial](https://docs.expo.dev/modules/config-plugin-and-native-module-tutorial/), **we could add support for `app.plugin.js` file for libraries and then declare them under `"plugins"` property in `app.json` the same way Expo is doing that with their libraries**:
```diff
{
    "name": "MyTestApp",
    "displayName": "MyTestApp",
+   "plugins": [
+      [
+        "expo-camera",
+         {
+           "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera."
+         }
+      ]
+   ]
}
```

You can find an example built for `react-native-image-picker` library in [Code samples](https://github.com/TMisiukiewicz/discussions-and-proposals/blob/rfc-config-plugins/proposals/0000-implement-config-plugins.md#code-samples) section.

If config plugins become a standard, we could provide support for it in [create-react-native-library](https://github.com/callstack/react-native-builder-bob).

## Potential benefits

### Improved updates
With this solution it seems like upgrading React Native would become much easier. Each time React Native gets upgraded, we can just copy-paste fresh iOS and Android templates from RN core and apply all the changes using config plugins. We could track if any manual changes were made in the native folders, and if yes, warn user about it and allow to upgrade only in an old way.

> For updating dependencies we could utilize [@rnx-kit/align-deps](https://microsoft.github.io/rnx-kit/docs/tools/align-deps)

### Support for other platforms
This approach would also make it possible to extend config plugins with other platforms, e.g. `react-native-windows`. Following the same pattern, a support for `windows` and `macos` properties could be added to `app.json` together with some custom mods for native files.

## Drawbacks

- some of the mods are operating on regexes, which makes them dangerous to use - especially when core developers are modifying template. Config plugins cannot cover every usage scenario safely. We suggest having some markers to allow modifying templates only between them. List of files modified based on regex: `MainActivity.java`, `MainApplication.java`, `build.gradle`, `settings.gradle`, `AppDelegate.m`
- might take some time for maintainers to adjust libraries to new approach
- bringing config plugins into core might generate scenarios where dependencies are making incompatible changes, so it might get hard to debug

## Alternatives

- installing Expo to the project and using `npx expo prebuild` - RN version is dependend on Expo SDK release cycle, however there is a `--template` flag which unlocks possibility of using any template with other version. This solution is not recommended by Expo because of making some undocumented assumptions about the template.
- integrating `@expo/config-plugins` with React Native CLI to unlock a path for developers to run `yarn add @expo/config-plugins` in their non-Expo projects and use it - we might end up with a misalignment between what parts of Expo Config Plugins we want to support in the CLI (everything that is not related to Expo) and what developers will be able to use in their projects (everything, even if it's Expo-specific). This may lead to many misunderstandings why some of the properties in `app.json` or some mods are not working.
- creating a new library based on Expo Config Plugins, exposing only stuff not related to Expo - it's not making Config Plugins a standard in the RN ecosystem, as it makes it totally optional to have it as a dependency in projects and libraries
- creating a brand new CLI supporting Config Plugins - we won't fix the major problem of the apps currently running on React Native CLI - easy RN upgrades. It would be another tool existing in the RN ecosystem, still requiring to add a new dependency to projects/libraries and working only when using it with specific tool.
- [`microsoft/react-native-test-app`](https://github.com/microsoft/react-native-test-app) - it already has config plugins implemented, but in our opinion config plugins need broader adoption

> RNTA does not have platform specific files checked in, they are provided by RNTA package or generated at a build time. However some of the files are in the repository, e.g. `Podfile` or `build.gradle`, but they have some configs applied directly from RNTA package.  

## Adoption strategy

- [ ] In close cooperation with Expo, upstreaming config plugins into RN core in a way that would help them easily adjust their tools.
- [ ] changes should be applied into `react-native-cli` to support config plugins.
- [ ] Both React Native and CLI should be covered with proper documentation:
- `reactnative.dev` with detailed description what config plugins are and how to use them - for both developers and library maintainers
- CLI's `README.md` file with detailed explanation of `prebuild` command and available flags
- [ ] community and maintainers should be informed about new API available and how it would benefit them
- [ ]  at the same time, [`create-react-native-library`](https://github.com/callstack/react-native-builder-bob) should be adjusted to support the new API.

## How we teach this

- React Native docs should be updated, describing well the new approach, Expo documentation would be very helpful here
- Post on React Native blog with explanation and some examples how it works and how it benefits developers and community in general
- Conferences talks, e.g. on React Native EU, to reach as large audience as possible.

From entry level developers perspective, using `app.json` file to handle configuration should be very easy. Creating custom plugins and using mods is a bit more difficult, but in general should be done by developers only when needed. Possibly the biggest audience here are library maintainers, with good understanding of the concept they will be able to adopt it into the libraries. Developers creating apps probably wouldn't have to worry about config plugins unless they need to create one.

## Unresolved questions
- What terminology should we use to distinguish approach with native folders kept out of repository and after prebuilding?
- How should we determine what to keep in `app.json` and `react-native.config.js`? Does [RFC: introduce reactNativeMetadata to package.json](https://github.com/react-native-community/discussions-and-proposals/pull/588/files) affect this proposal?
- Could we utilize some work already done on [microsoft/react-native-test-app](https://github.com/microsoft/react-native-test-app)?
