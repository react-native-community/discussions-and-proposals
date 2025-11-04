---
title: Standardized Brownfield Integration APIs for React Native (Android)
author:
- Hur Ali
date: 4th Nov 2025
---

# RFC0000: Standardized Brownfield Integration APIs for React Native (Android)

## Summary

This proposal introduces official abstractions and APIs in React Native Core to simplify brownfield (hybrid) integrations on Android. These APIs aim to standardize how existing native Android apps embed React Native screens and manage lifecycle, back press, and runtime initialization — all without requiring extensive manual setup or boilerplate code.

## Basic example

### Example: Embedding React Native in an existing Android app

```kotlin
val defaultReactFragment = DefaultReactFragment(activity, launchOptions) {
  override fun moduleName(): String = "RNProfileEntry"

  override fun onHostResume() {
    val defaultBackHandler = DefaultHardwareBackBtnHandler {
      activity.onBackPressedDispatcher.onBackPressed()
    }
    return reactHost.onHostResume(activity, defaultBackHandler)
  }
}

supportFragmentManager
  .beginTransaction()
  .add(R.id.container, defaultReactFragment)
  .commit()
```

This demonstrates how native developers can present a React Native screen with minimal setup using the proposed DefaultReactFragment abstraction.

## Motivation

Brownfield adoption of React Native is a common real-world scenario where existing native apps incrementally embed React Native screens or features. While this is achievable today through Core classes, community plugins, and manual setup, there is no single, consistent API surface designed specifically for brownfield integration.

This creates friction for teams that want to:

- Embed React Native without restructuring their entire Android app.

- Avoid scattering React Native–specific interfaces and lifecycle handling throughout native codebases.

- Provide a smooth developer experience for both native and React Native engineers.

> This is often the first experience native developers have with React Native, it must feel straightforward and consistent.

By exposing a small set of official classes dedicated to brownfield usage, React Native Core can standardize integrations, reduce boilerplate, and improve long-term maintainability.

## Detailed design

### Current State of Brownfield Integration

There are two primary ways to embed React Native UI into an Android app:

1. Configure React Native directly in the Android app

   - Configure a Node environment within your Android project.

   - Add react-android and hermes-android dependencies.

   - Update build.gradle and settings.gradle.

   - Extend ReactActivity and implement ReactApplication.

   - Use these to launch React Native UI directly.

2. Configure React Native in an Android library

   - Create a library module inside a React Native app alongside the app module.

   - Add react-android and hermes-android as api dependencies.

   - Use community plugins (e.g., brownfield-gradle-plugin) to link native modules, assets, and JNI libs.

   - Embed React Native UI from this library into the native app.

Both approaches are valid and come with trade-offs — the first offers direct integration, while the second encourages modularity and reuse.

<hr />
<!-- ### Pain Points -->


### 1. Initializing the React Native runtime

Developers can achieve this by subclassing their `MainApplication` with `ReactApplication`.

```kotlin
class MainApplication : Application(), ReactApplication {

  override val reactHost: ReactHost by lazy {
    getDefaultReactHost(
      context = applicationContext,
      packageList =
        PackageList(this).packages.apply {
          // Packages that cannot be autolinked yet can be added manually here, for example:
          // add(MyReactNativePackage())
        },
      jsBundleAssetPath = "index",
      jsMainModulePath = "index.android.bundle",
      useDevSupport = BuildConfig.DEBUG
    )
  }

  override fun onCreate() {
    super.onCreate()
    loadReactNative(this)
  }
}
```

If for some reasons, the developers can not subclass their `MainApplication` with `ReactApplication`, they can manually create the `reactHost` and store a reference to it.

```kotlin
val reactHost: ReactHost by lazy {
  getDefaultReactHost(
    context = application,
    packageList = PackageList(this).packages,
    jsBundleAssetPath = "index",
    jsMainModulePath = "index.android.bundle",
    useDevSupport = BuildConfig.DEBUG
  )
}

// invoke this function in MainApplication.onCreate
loadReactNative(this)
```

This API is in a good state and we do not propose any changes for this part.


### 2. Presenting React Native UI

Current options:

- ReactDelegate – good for embedding smaller components, but limited lifecycle handling.

```kotlin
val reactDelegate = ReactDelegate(activity, reactHost!!, componentName, initialParams)

reactDelegate.loadApp()
return reactDelegate.reactRootView as FrameLayout
```

- ReactFragment – wraps ReactDelegate, better for full-screen flows.

```kotlin
val reactNativeFragment = ReactFragment.Builder()
		.setComponentName("HelloWorld")
		.setLaunchOptions(Bundle().apply { putString("message", "my value") })
		.build()

supportFragmentManager
		.beginTransaction()
		.add(R.id.react_native_fragment, reactNativeFragment)
		.commit()
```

However, both require the hosting Activity to implement `DefaultHardwareBackBtnHandler`, introducing React Native–specific imports into native activities. This however, depends on the trade-off the native team is willing to take. If they are okay with colocating the RN internals in their Native Apps, this is not a problem for them. However, there are some teams who do not wish to take this path and want to abstract all of the RN internals in the AAR, this produces a problem for them.

> There is another way to present RN UIs by using ReactActivity. While this is a good option to have, not all teams would want to use it.

### 3. Hardware Back Button Behavior

In mixed stacks, pressing back often skips the JS navigation stack entirely, e.g.:

```
Native App → MainActivity → NativeHomeFragment → ReactFragment("Settings") → JSscreenA → JSscreenB
```


Back press on JSscreenB can incorrectly return to NativeHomeFragment. Workarounds exist (e.g., intercepting OnBackPressedDispatcher), but Core support is needed for a clean developer experience.

<hr />

## Proposal:

Introduce official brownfield abstractions in React Native Core that mirror `DefaultReactHost` but for embedding UI.

### 1. DefaultReactDelegate (new)
```kotlin
val defaultReactDelegate = DefaultReactDelegate(activity, reactHost, launchOptions) {
  override fun moduleName(): String = "RNProfileEntry"

  override fun onHostResume() {
    val defaultBackHandler = DefaultHardwareBackBtnHandler {
      activity.onBackPressedDispatcher.onBackPressed()
    }
    
    return reactHost.onHostResume(activity, defaultBackHandler)
  }
}

return defaultReactDelegate.reactRootView // returns the RN UI embedded
```

### 2. DefaultReactFragment (new)
```kotlin
val defaultReactFragment = DefaultReactFragment(activity, launchOptions) {
  override fun moduleName(): String = "RNProfileEntry"
  
  override fun onHostResume() {
   val defaultBackHandler = DefaultHardwareBackBtnHandler {
     activity.onBackPressedDispatcher.onBackPressed()
   }
    
    return reactHost.onHostResume(activity, defaultBackHandler)
  }
}

supportFragmentManager
  .beginTransaction()
  .add(R.id.container, defaultReactFragment)
  .commit()
```

These options will allow the developers to present their RN UI without requiring the host `Activity` to implement `DefaultHardwareBackBtnHandler`. This also lays the foundation to handle the JS stack back navigation behavior by allowing developers to specify their `defaultBackHandler`.

### 2. Improved Back Press Integration (new)

With the above set, the developer can provide and handle the JS stack back navigation according to their needs and they can provide the defaultBackHandler, which will be executed on the last RN screen popped.

```kotlin
val backPressedCallback: OnBackPressedCallback = object : OnBackPressedCallback(true) {
   override fun handleOnBackPressed() {
     // invoked for JS stack back navigation
     reactDelegate.onBackPressed()
   }
}

activity.onBackPressedDispatcher.addCallback(backPressedCallback)

val defaultReactFragment = DefaultReactFragment(activity, launchOptions) {
  override fun moduleName(): String = "RNProfileEntry"
  
  override fun onHostResume() {
   // invoked on the last RN screen
   val defaultBackHandler = DefaultHardwareBackBtnHandler {
     backPressedCallback.isEnabled = false
     activity.onBackPressedDispatcher.onBackPressed()
   }
    
    return reactHost.onHostResume(activity, defaultBackHandler)
  }
}
```

To summarize, these APIs can be used by both options for brownfield integrations, one where devs directly integrate react-native with the native App and other where they opt for configuring RN as an android library.

- DefaultReactDelegate (new)
- DefaultReactFragment (new)
  - Maybe we can rename the ReactFragment and confirm it to the proposal in this RFC

## Drawbacks

Why should we _not_ do this? Please consider:

- Increased API surface area in React Native Core.
- Potentially requires making `ReactDelegate.onHostResume` non-final. This is required to improve the back press integration.
- If introducing `DefaultReactFragment` we may need to drop `ReactFragment` as both serve the same purpose. We can either add the changes from this proposal directly to `ReactFragment` OR we can introduce `DefaultReactFragment` and mark `ReactFragment` as deprecated and remove in the future releases.

## Alternatives

- Continue using community-maintained plugins such as react-native-brownfield.
- Continue using the documentation or templates.
  - https://www.rockjs.dev/docs/brownfield/android
  - https://reactnative.dev/docs/integration-with-existing-apps

> However, these alternatives are dependent on the RN Core APIs and without brownfield dedicated APIs, they can produce a knock-on effect. One such example is the changes introduced from 0.80 onwards. Apart from it, some of these alternatives require more boilerplate from the native developer if they are opting for direct integration.


## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

- Update the official docs to cover showing RN UI using `ReactActivity`, `DefaultReactFragment` and `DefaultReactDelegate`.
  - https://reactnative.dev/docs/integration-with-existing-apps

## How we teach this

- The docs can contain both the approaches, direct RN integration or library based RN inegration. We already have the direct RN integration docs. However, they can use some tweaks. For example, apart from showing how to initialize `reactHost` by subclassing `ReactApplication`, we can also show a way without a need to subclass `ReactApplication`.
- Update the official docs to cover showing RN UI using `ReactActivity`, `DefaultReactFragment` and `DefaultReactDelegate`.
  - https://reactnative.dev/docs/integration-with-existing-apps

- Reference these and other APIs as the preferred approach for embedding RN into native Android apps.
  - ReactActivity
  - DefaultReactDelegate
  - DefaultReactFragment(or ReactFragment)
  - getDefaultReactHost

## Unresolved questions

Should Core expose an API to preload JS bundles for faster first render?

> Historically, this was possible with `reactInstanceManager`.createReactContextInBackground(). Since `ReactNativeHost` is deprecated post–v0.82, there’s no clear alternative. Could reactHost.reload("reason") serve this purpose?

<hr />