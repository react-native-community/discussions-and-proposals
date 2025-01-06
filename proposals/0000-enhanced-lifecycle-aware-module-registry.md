---
title: Enhanced lifecycle-aware module registry
author:
    - Matin Zadeh Dolatabad
date: 2025-01-06
---

# RFC0000: Enhanced lifecycle-aware module registry

## Summary

This RFC proposes a framework to enhance the module registry in React Native by introducing a lifecycle-aware system for native modules. The goal is to address the current gap in handling application lifecycle events, similar to Flutter's `FlutterApplicationLifeCycleDelegate` on iOS and `Application.ActivityLifecycleCallbacks` on Android. The design enables seamless integration of native modules with application lifecycle events across iOS and Android platforms. There is also Expo Modules Core that handles this, but React Native does not have this by default and it requires Expo to be used in such cases.

## Motivation

React Native lacks a comprehensive and standardized approach for native module lifecycle management. Unlike Flutter, where plugins can hook into application lifecycle events via a well-defined protocol, React Native requires developers to implement custom solutions for such behavior. This limitation results in increased boilerplate code, fragmented implementations, and an inconsistent developer experience.

### Key Challenges Addressed:

- **Limited Module Registry:** React Native's module registry doesn't natively support lifecycle awareness. On Android, React Native provides a `LifecycleEventListener` that partially addresses this by allowing modules to listen to events like `onHostResume`, `onHostPause`, and `onHostDestroy`. However, there is no support for early lifecycle events such as `onCreate` or `onTerminate`. On iOS, there is no equivalent lifecycle support, leading to significant gaps in managing the full application lifecycle effectively across platforms.
- **Manual Lifecycle Handling:** Developers must manually wire up lifecycle events for each native module.
- **Inconsistency Across Platforms:** There is no unified approach for managing lifecycle events across iOS and Android except `Expo Modules API`.

## Detailed Design

The proposed solution introduces a `LifecycleAwareModule` interface and a centralized lifecycle manager for React Native. This framework would:

1. Allow native modules to register as lifecycle-aware.
2. Notify registered modules of key application lifecycle events.
3. Provide a consistent API across platforms.

### iOS Implementation

Define a protocol for lifecycle-aware modules:

```objc
@protocol RNLifecycleAwareModule <NSObject>
@optional
- (void)applicationDidFinishLaunching:(UIApplication *)application;
- (void)applicationWillResignActive:(UIApplication *)application;
- (void)applicationDidEnterBackground:(UIApplication *)application;
- (void)applicationWillEnterForeground:(UIApplication *)application;
- (void)applicationDidBecomeActive:(UIApplication *)application;
- (void)applicationWillTerminate:(UIApplication *)application;
@end
```

Create a `RNLifecycleManager` to manage registered modules:

```objc
@interface RNLifecycleManager : NSObject
+ (instancetype)sharedInstance;
- (void)registerModule:(id<RNLifecycleAwareModule>)module;
- (void)notifyModulesWithSelector:(SEL)selector application:(UIApplication *)application;
@end
```

Hook into `AppDelegate`:

```objc
@implementation AppDelegate
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [[RNLifecycleManager sharedInstance] notifyModulesWithSelector:@selector(applicationDidFinishLaunching:) application:application];
    return YES;
}
@end
```

### Android Implementation

Define an interface for lifecycle-aware modules:

```kotlin
interface RNLifecycleAwareModule {
    fun onCreate(application: Application)
    fun onResume()
    fun onPause()
    fun onStop()
    fun onDestroy()
}
```

Create a `LifecycleManager` to manage registered modules:

```kotlin
object LifecycleManager {
    private val modules = mutableListOf<RNLifecycleAwareModule>()

    fun registerModule(module: RNLifecycleAwareModule) {
        modules.add(module)
    }

    fun notify(event: (RNLifecycleAwareModule) -> Unit) {
        modules.forEach { event(it) }
    }
}
```

Hook into `Application`:

```kotlin
class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        LifecycleManager.notify { it.onCreate(this) }
    }

    override fun onTerminate() {
        super.onTerminate()
        LifecycleManager.notify { it.onDestroy() }
    }
}
```

### Usage Inside Libraries

Libraries can leverage the lifecycle-aware module registry to handle lifecycle events seamlessly. For instance:

#### iOS Example

```objc
@interface MyModule : NSObject <RNLifecycleAwareModule>
@end

@implementation MyModule

- (void)applicationDidFinishLaunching:(UIApplication *)application {
    // Initialize resources
}

- (void)applicationDidEnterBackground:(UIApplication *)application {
    // Save state
}

@end
```

#### Android Example

```kotlin
class MyModule : RNLifecycleAwareModule {
    override fun onCreate(application: Application) {
        // Initialize resources
    }

    override fun onPause() {
        // Save state
    }
}
```

By implementing the `RNLifecycleAwareModule` interface, library developers can ensure that their modules respond appropriately to lifecycle events without requiring additional setup from the application developers.

However the design details here are just a demonstration and the actual implementation may vary. This implementation ca be baked into the current interface or a new interface can be introduced.

## Drawbacks

- **Backward Compatibility:** Existing modules will need updates to implement the lifecycle-aware interface.
- **Performance Overhead:** Centralized lifecycle management may introduce performance overhead, especially with many registered modules.
- **Potential Conflicts:** Modules that handle lifecycle events internally may conflict with the centralized lifecycle manager.

## Alternatives

### Option 1

Use third-party libraries for lifecycle management, but this lacks standardization and official support.

### Option 2

Continue with the current approach, leaving lifecycle management to individual frameworks/developers.

## Adoption Strategy

- Introduce as an optional feature in a minor React Native release.
- Provide detailed documentation and examples.
- Encourage library authors to adopt the lifecycle-aware interface for their modules.

## How We Teach This

- Update official documentation with lifecycle-aware module examples.
- Provide tutorials and migration guides for developers.
- Offer webinars or workshops to explain the new system.

## Unresolved Questions

- How to handle potential performance overhead with many registered modules?
- How to ensure compatibility with existing React Native libraries?
