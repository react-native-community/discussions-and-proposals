---
title: implement setBundleSource to customise the Bundle URL
author: Jakub Piasecki
date: 01.08.2025
---

# RFC0000: Implement setBundleSource to customise the Bundle source.

## Summary
In this RFC, we propose adding a `setBundleSource` method that would allow us to change the bundle dynamically and without the need to modify `MainApplication` or `AppDelegate`.

This API is targeted to be utilized by frameworks.
## Motivation
A public API allowing to change the bundle URL at runtime, without the need to modify the application files, would significantly reduce the effort it takes to update the React Native version in React Native Frameworks. It would also make supporting multiple versions of React Native by a single framework version easier, thanks to API stability.

Currently, on Android, there is no way to customize the instance of `PackagerConnectionSettings`, as all references to it are read-only ([1](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/modules/debug/interfaces/DeveloperSettings.kt#L15), [2](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/devsupport/interfaces/DevSupportManager.kt#L27), [3](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/runtime/ReactHostImpl.kt#L100)). It is possible to change just the `debugServerHost` on the existing `PackagerConnectionSettings` instance, however and the set value is persisted on disk. Customization of the query passed to the packager is also limited.

On iOS, `RCTPackagerConnection` relies only on `RCTBundleURLProvider` as the source of the bundler location, which is a singleton. There is no way to customize the bundler location for a single `RCTHost`.

## Detailed design
### Android
We propose to implement two new methods on `ReactHost`, which should be able to handle local files as well as remote connections:
```kotlin
public fun setBundleSource(filePath: string)

public fun setBundleSource(debugServerHost: string, moduleName: string, queryBuilder: (Map<String, String>) -> Map<String, String> = { it })
```
On Android, the bundle is always loaded from file - when loading from Metro, the bundle is first saved on disk and then loaded.

We propose adding a new property to the [`DevSupportManager`](https://github.com/facebook/react-native/blob/b02251e7f5c147296fab93c1ae613d27400cec92/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/devsupport/interfaces/DevSupportManager.kt#L31): `customBundleFilePath: String?`. This property would be null by default, but could be modified from the new `setBundleSource` method. If defined, this setting would take priority over other methods of loading bundle inside [`jsBundleLoader`](https://github.com/facebook/react-native/blob/b02251e7f5c147296fab93c1ae613d27400cec92/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/runtime/ReactHostImpl.kt#L1045) task, and the bundle would be loaded from the specified path instead of the transient file. This would apply to both Debug and Release modes.

Customizing the URL from which the bundle is loaded from Metro could be done by allowing [`packagerConnectionSettings`](https://github.com/facebook/react-native/blob/b02251e7f5c147296fab93c1ae613d27400cec92/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/modules/debug/interfaces/DeveloperSettings.kt#L15) instance to be changed, and [`jsAppBundleName`](https://github.com/facebook/react-native/blob/b02251e7f5c147296fab93c1ae613d27400cec92/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/devsupport/DevSupportManagerBase.kt#L81) to be modified. `setBundleSource` method could be updating the module name that should be bundled and replacing the `PackagerConnectionSettings` with a new instance that would point to the set host, and it would allow to customize the query string based on the provided `queryBuilder`. This function would accept a map of default key-value pairs, which would have been used to build the bundle URL, and return a new, potentially changed one. Calling this implementation of `setBundleSource` would need to clear the potential value of `customBundleFilePath` since it would have a higher priority than the bundle connection.

This would require updating [`PackagerConnectionSettings`](https://github.com/facebook/react-native/blob/84472d9ebceff09d329876270074a89b90a6fb87/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/packagerconnection/PackagerConnectionSettings.kt#L18) with a new method that would allow for manipulating all query arguments instead of only defining new ones:
```kotlin
private var _packagerOptionsUpdater: Map<String, String>) -> Map<String, String> = { it }

public fun setPackagerOptionsUpdater(updater: (Map<String, String>) -> Map<String, String>) {  
  _packagerOptionsUpdater = updater
}

public fun updatePackagerOptions(options: Map<String, String>) = _packagerOptionsUpdater(options)
```
Those APIs would co-exist to maintain backwards-compatibility but [the old one](https://github.com/facebook/react-native/blob/84472d9ebceff09d329876270074a89b90a6fb87/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/packagerconnection/PackagerConnectionSettings.kt#L53-L58) could be deprecated.

[`createBundleURL`](https://github.com/facebook/react-native/blob/f21a89078c69a71a9ff5fd6ba77b38ec9ed272f4/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/devsupport/DevServerHelper.kt#L259-L288) would need to be updated to generate a map with all the key-value pairs used for the default query string, along with the additional ones. Then pass it through the updater function before using it to build the query string.

After `setBundleSource` is called [`reactLifecycleStateManager`](  private val reactLifecycleStateManager = ReactLifecycleStateManager(bridgelessReactStateTracker) would be checked. If in `BEFORE_CREATE` state and the task responsible for creating the `ReactInstance` hasn't started yet, it could stop at this point. If that condition isn't met, the reload should be triggered to load the new bundle. This could reuse already existing [`reload`](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/runtime/ReactHostImpl.kt#L372) method.

### iOS
There already exists an API for replacing the bundle URL - [overriding `bundleURL` in the RCTReactNativeFactoryDelegate](https://github.com/facebook/react-native/blob/504cf3e9330285ba8995dd938756d2ea13ffa28d/packages/react-native/Libraries/AppDelegate/RCTReactNativeFactory.mm#L108-L116), however, this doesn't affect the packager connection in any way. There is [`setJsLocation`](https://github.com/facebook/react-native/blob/504cf3e9330285ba8995dd938756d2ea13ffa28d/packages/react-native/React/Base/RCTBundleURLProvider.mm#L423-L426), however, it persists the value and is a global setting, not scoped to a specific host.

Both, [`RCTBundleURLProvider`](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/React/Base/RCTBundleURLProvider.mm#L443-L451) and [`RCTPackagerConnection`](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/React/DevSupport/RCTPackagerConnection.mm#L55-L63) are singletons. We propose to update the `RCTPackagerConnection` in two significant ways:

First, associate it with the `RCTDevSettings` instance, where the current singleton is accessed. Relying on a private instance, [instead of a shared one](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/React/CoreModules/RCTDevSettings.mm#L182), would allow for safer and scoped customization.

Second, make it rely on [`RCTBundleManager`](https://github.com/facebook/react-native/blob/504cf3e9330285ba8995dd938756d2ea13ffa28d/packages/react-native/ReactCommon/react/runtime/platform/ios/ReactCommon/RCTHost.mm#L117), specific to a given `RCTHost`. Relevant instance of `RCTBundleManager` is already accessible from [`RCTDevSettings`](https://github.com/facebook/react-native/blob/504cf3e9330285ba8995dd938756d2ea13ffa28d/packages/react-native/React/CoreModules/RCTDevSettings.mm#L135). `RCTBundleManager` would need to be extended itself with an optional configuration field:
```objective-c
@property (nonatomic, readonly, nullable) RCTCustomBundleConfiguration *customBundleConfig;
```
```objective-c
@interface RCTCustomBundleConfiguration

@property (nonatomic, readonly, nullable) NSURL *bundleFilePath;

@property (nonatomic, readonly, nullable) NSString *packagerServerScheme;

@property (nonatomic, readonly, nullable) NSString *packagerServerHost;

- (NSURL *)getBundleURL:(NSMutableArray<NSURLQueryItem *> *)query;

@end
```
Only one of `bundleFilePath` and `(packagerServerScheme, packagerServerHost)` would be allowed to be set. This configuration object would be set on the `RCTReactNativeFactory`, which would pass it down to `RCTRootViewFactory`, which would pass it down to `RCTHost`. `RCTHost` would need to be modified to accept the new configuration object and set it to the instance of [`RCTBundleManager`](https://github.com/facebook/react-native/blob/504cf3e9330285ba8995dd938756d2ea13ffa28d/packages/react-native/ReactCommon/react/runtime/platform/ios/ReactCommon/RCTHost.mm#L166).

The logic responsible for building the URL query from [`RCTBundleURLProvider::jsBundleURLForBundleRoot`](https://github.com/facebook/react-native/blob/f21a89078c69a71a9ff5fd6ba77b38ec9ed272f4/packages/react-native/React/Base/RCTBundleURLProvider.mm#L305-L352) would be extracted to a separate method - `createJsBundleURLQuery`. It would return the default query, which would then be passed to the `getBundleURL` method in the configuration object, where it could be modified by the frameworks before returning the final URL.

`RCTPackagerConnection` would need to be updated to connect to the host from the configuration object when it's set.

This will shift the responsibility from managing the packager connection from frameworks to React Native and solving the issue of multiple connections (which can happen when trying to work around the API limitations currently, by manipulating the WebSocket connection directly) to the packager for the same instance of `RCTHost`.
## Drawbacks
Given that this is a new API that is being added only for frameworks, we don't see drawbacks in implementing it, aside from the need to maintain it.
## Alternatives
Expo, for example, is using reflection on [Android to replace the instance of `PackagerConnectionSettings` being used](https://github.com/expo/expo/blob/b90fea77e87bb03f3c65ceeaf8913c96a7c26aff/packages/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/helpers/DevLauncherReactUtils.kt#L114-L141). [On iOS, KVC is used to restart the socket connected to the packager with the new URL](https://github.com/expo/expo/blob/9770def34c359ee20145cbcc031c1d2322fec4e0/packages/expo-dev-launcher/ios/Unsafe/RCTPackagerConnection%2BEXDevLauncherPackagerConnectionInterceptor.m#L40-L69).

Another approach would be to rely on existing APIs, allowing to accomplish similar results:
- On Android, it’s possible to set [`debugServerHost`](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/packagerconnection/PackagerConnectionSettings.kt#L24), but due to persistence, it could result in unexpected behaviors when working with multiple React hosts with different bundler URLs.
- On iOS, [`RCTBundleURLProvider`](https://github.com/facebook/react-native/blob/5f3eccb04fe89f492c695dfad9496473874b3658/packages/react-native/React/Base/RCTBundleURLProvider.mm) serves as the source of truth when it comes to accessing the packager location. It’s possible to modify this API, but since it's public and [relied upon in OSS](https://github.com/search?q=RCTBundleURLProvider&type=code), it would have to be a breaking change.
## Adoption strategy / How we teach this
As a part of the implementation, documentation for the new API will be provided with example usages to showcase how it works and explain the use cases.
## Unresolved questions
## Acknowledgements
Many thanks to Expo for raising the problem and drafting the initial RFC.