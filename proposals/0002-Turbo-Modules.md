# RFC0002: Turbo Modules ™ 
_Its blazing fast_

## Summary

React Native uses [Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios) as a way for JavaScript  to invoke functions in Java/ObjC. Listed unde the API section in the [React Native documentation](https://facebook.github.io/react-native/docs/0.56/getting-started), popular native modules include Geolocation, Device Information, Async Storage, etc.
This proposal discusses ways to speed up the way Native Modules are initialized and invoked from JavaScript.

## Motivation

This section lists the problems with the current state. Though Android is used as the example, similar issues exist in iOS also. 

1. Native Modules are specified in [a package](https://github.com/facebook/react-native/blob/1e8f3b11027fe0a7514b4fc97d0798d3c64bc895/local-cli/link/__fixtures__/android/0.20/MainActivity.java#L37) and are eagerly initialized. The startup time of React Native increases with the number of Native Modules, even if some of those Native Modules are never used. Native Modules can be made lazy by using [LazyReactPackage](https://github.com/facebook/react-native/blob/42146a7a4ad992a3597e07ead3aafdc36d58ac26/ReactAndroid/src/main/java/com/facebook/react/LazyReactPackage.java) but this does not work in the Open Source repo since  work since the annotation processor called  [ReactModuleSpecProcessor](https://github.com/facebook/react-native/blob/42146a7a4ad992a3597e07ead3aafdc36d58ac26/ReactAndroid/src/main/java/com/facebook/react/module/processing/ReactModuleSpecProcessor.java) does not work with Gradle yet. Note that there was [a PR to solve this](https://github.com/facebook/react-native/pull/10084), but it was not merged.  

2. There is no simple way to check if the Native Modules that JavaScript calls are actually included in the native app. With over the air updates, there is no easy way to check if a newer version of JavaScript calls the right method with the correct set of arguments in Native Module. 

3. Native Modules are always singleton and their lifecycle is typically tied to the lifetime of the bridge. This issue is compounded in brownfield apps where React Native bridge may be started and shut down multiple times.

4. During the startup process, Native Modules are typically specified in [multiple packages](https://github.com/facebook/react-native/blob/b938cd524a20c239a5d67e4a1150cd19e00e45ba/ReactAndroid/src/main/java/com/facebook/react/CompositeReactPackage.java). We then [iterate](https://github.com/facebook/react-native/blob/407e033b34b6afa0ea96ed72f16cd164d572e911/ReactAndroid/src/main/java/com/facebook/react/ReactInstanceManager.java#L1132) over these list [multiple times](https://github.com/facebook/react-native/blob/617e25d9b5cb10cfc6842eca62ff22d39eefcf7b/ReactAndroid/src/main/java/com/facebook/react/bridge/NativeModuleRegistry.java#L46) before we finally give the bridge, a [list of Native Modules](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/CatalystInstanceImpl.java#L124) to the bridge. This does not need to happen at runtime. 

5. The actual methods and constants of a Native Module are [computed during runtime](https://github.com/facebook/react-native/blob/master/ReactCommon/cxxreact/ModuleRegistry.cpp#L81) using [reflection](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/JavaModuleWrapper.java#L75). This can also be done using build time. The individual method calls themselves are sent over a message queue. 

## Background

### JavaScript Interface
The communication between JavaScript and the VM is being standardized using a JavaScript Interface (__JSI__). [Fabric](https://github.com/react-native-community/discussions-and-proposals/issues/4) uses JSI to [expose methods](https://github.com/facebook/react-native/blob/5d9326be29be8688f2238c72c18a9037f983c77d/ReactAndroid/src/main/java/com/facebook/react/fabric/jsc/jni/FabricJSCBinding.cpp#L264) to JavaScript code. Using JSI, JavaScript can hold reference to C++ Host Objects and invoke methods on them. 

### Native Modules 
 > [TODO] - A detailed section on how native modules work today. 

## Detailed design

## Defining Native Modules
Internally, we use flow types to define the Native Modules and their method and signatures (also called "shape of Native Modules"). This is then used to generate Java and ObjC classes with method stubs with matching signatures, which can be extended for the actual implementation. We plan to now generate C++ classes with the same shape, that can then be used to call into the Java/ObjC implementations. 

## Initial Processing
All initial processing will be removed as C++ now knows the shape of Native Modules. A proxy similar to the [NativeModuleProxy](https://github.com/facebook/react-native/blob/da7873563bff945086a70306cc25fa4c048bb84b/ReactCommon/cxxreact/JSCExecutor.cpp#L141) will be defined in C++ and [exposed in JavaScript](https://github.com/facebook/react-native/blob/a93e281428f47ba355ba79cda3e98ca7da677636/Libraries/BatchedBridge/NativeModules.js#L152). The initialization can typically happen when the RootView is initialized, but once we are independent of the bridge, this can also happen at other times. 

## Invoking methods on Native Module
When the JavaScript code like `require('NativeModule').DeviceInfoModule` is called, the getter on the proxy is invoked. It would return a reference to object generated in C++ with the shape of the Native Module. Method calls on this reference would simply call over to Java or ObjC. The actual call may also need to have checks to ensure the type and count of arguments. 

> [TODO] - A work in process.

## Open Questions
1. How would the C++ object corresponding to the shape of a Native Module know the Java class (for example com.facebook.react.nativemodule.DeviceInfo) that it needs to call. Does this need to be a switch/case or a map in Java/ObjC that is constructed manually ? Would this be done using BUCK dependencies ? 
2. Today, native modules usually take `ReactApplicationContext` as a parameter when they are constructed, but they can also be constructed in other ways like using a builder, or multiple parameters in the constructor. How would this be enforced ? 
3. How would the C++ code gen happen in Open Source, where BUCK is not used ? 
4. Since the generated C++ host objects have the same lifetime of the JS scopes they are referred in, what is the API to tell Java/ObjC when these objects are deallocated ? 
5. ReactModule options like `OnBatchCompleteListener` and `hasEagerInitialization` will be removed. Are they needed anymore, if yes - whats the alternative ? 
6. This method allows Native Modules to not be singletons. What would this new pattern look like ? Would this lead to memory leaks ? 

## Backward Compatiability
1. It looks like existing native modules do not have to change since they are usually singletons. 
