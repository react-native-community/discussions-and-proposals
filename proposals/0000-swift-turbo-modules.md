---
title: Swift Turbo Modules
author:
- Oskar Kwasniewski 
- Riccardo Cipolleschi
date: 12-08-2024 
---

# RFC0000: Swift Turbo Modules

## Summary

This RFC aims to allow developers to write Turbo Modules using Swift. This will allow the usage of more modern language making maintenance of native modules easier and more accessible. 

## Motivation

The primary motivations for introducing Swift Turbo Modules are:
- Enhance developer experience for iOS developers working with React Native
- Allow the use of more modern language
- Lower the entry barrier to write a native turbo module


## Detailed design

One of the reasons why we can't adopt Swift in TurboModules is the contamination of C++ types ending up in user-space.

### Current Situation
The interfaces we generate for an Objective-C turbomodules have this shape:
```objc
@protocol NativeMyTurboModuleSpec <RCTBridgeModule, RCTTurboModule>
@end
```

The `RCTTurboModule` protocol requires the conforming object to implement a method with this signature:
```objc
 - (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
       (const facebook::react::ObjCTurboModule::InitParams &)params;
```

The signature of this method contains two types from C++:

* `std::shared_ptr<facebook::react::TurboModule>`
* `facebook::react::ObjCTurboModule::InitParams`

### Solution

The idea is to wrap the `getTurboModule` invocation in a `ModuleFactory` object.

The `TurboModuleWrapper` object is a base class that is supposed to be extended by a companion object for TurboModules. The base class has this interface:
```objc
@interface TurboModuleWrapper: NSObject
- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params;
@end
```
And the base class implementation just fails, as we don't want for it to be used directly.

Then, the `RCTTurboModule` interface can ask each TM to actually return an implementation of the `TurboModuleWrapper` rather then the actual `TurboModule`.

The `TurboModuleWrapper` is a pure Objective-C class, so it will work seamlessly with ObjectiveC and Swift.

So, now, the public interface of a TurboModule will only have pure objc entries and no C++ code.

When it comes to the implementation, the user-defined TurboModule  won't have to deal with any C++ code:

```objc
@implementation MyTurboModule
- (TurboModuleWrapper *)moduleFactory
{
return [[MyTurboModuleWrapper alloc] init];
}

// ... rest of the TM methods ...

@end
```

The `MyTurboModuleWrapper` implementation can be Codegenerated! 

We don't even need to ask our users to write that code themselves, as we have all the informations we need in the Codegen already.


The implementation will look like (note that `<UserDefinedName>` is something we get from Codegen):
```objc
// In the .h file

@interface Native<UserDefinedName>Wrapper: TurboModuleWrapper
@end

// In the .mm file

@implementation Native<UserDefinedName>Wrapper

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
   (const facebook::react::ObjCTurboModule::InitParams &)params;
{
  return std::make_shared<facebook::react::Native<UserDefinedName>SpecJSI>(params);
}

@end
```


The user-defined TurboModule has already access to this file, and so the switch from `getTurboModule` to `getWrapper` doesn't require any additional includes.


Finally, we will have to update the `RCTTurboModuleManager` to take this new object into consideration. So we have to modify the provide Turbomodule with the following code
```objc
  // Step 2e: Return an exact sub-class of ObjC TurboModule
  std::shared_ptr<facebook::react::TurboModule> turboModule = nullptr;
  
  if ([module respondsToSelector:@selector(getTurboModule:)]) {
    turboModule = [module getTurboModule:params];
  } else if ([module respondsToSelector:@selector(getWrapper)]) {
    auto wrapper = [module getWrapper];
    turboModule = [wrapper getTurboModule:params];
  }
```


Additionally we need to make sure that `ReactCodegen` module is compatible with importing to Swift. I did a small test and adding few ifdefs to React_Codegen headers allows us to use Swift.

```swift
import protocol ReactCodegen.NativeSwiftTestLibrarySpec
import protocol ReactCodegen.TurboModuleWrapper
import class ReactCodegen.NativeSwiftTestLibraryWrapper

@objc public class SwiftTestLibrary: NSObject, NativeSwiftTestLibrarySpec {
  @objc public func multiply(_ a: Double, b: Double) -> NSNumber! {
    return a * b as NSNumber
  }
  
  @objc public static func moduleName() -> String! {
    return "SwiftTestLibrary"
  }
 
  @objc public func getWrapper() -> (any TurboModuleWrapper)! {
    return NativeSwiftTestLibraryWrapper()
  }
}


public func SwiftTestLibraryCls() -> AnyClass {
    return SwiftTestLibrary.self
}
```

Here is a POC implementation of the proposal: https://github.com/okwasniewski/react-native/commit/93b21d1a2e5769924ae1913e912e94296a92f3d8 (using @protocol).

## Drawbacks

- Additional complexity in codegen
- Setup Swift CI/CD to test if there are no regressions breaking swift builds

## Alternatives

- Use Objective-C for all native modules

- **Using a protocol for the wrapper.**
	The pro of this is that we don't have an empty implementation for the TurboModuleWrapper object. 
	The cons are various:
	* The `TurboModule` itself can't adopt the `TurboModuleWrapper` protocol as, otherwise, the C++ signature will come back to the public API of the I don't think this will work. 
	* We can't codegen the default implementation for the `getTurboModule` as we won't have the base class for the protocol.
	* We could use a protocol and create a companion object in the codegen which extends the protocol and it is returned by the TurboModule. This solution works, but adds a bit of ceremonies to the base implementation above.

- **Using a custom base class for each TurboModule**
	The only pro of this approach is that we can remove a few lines from the definition of every TurboModule. 
	The con of this approach are:
	* The C++ code remains in the public API of the TurboModule
	* We are creating a deeper inheritance chain which usually should be avoided.
	* We are asking to all the users to inherit from a different base class. This is hard to make backward compatible.

## Adoption strategy

- Introduce as an experimental feature in a future React Native release
- Provide comprehensive documentation and migration guides
- This proposal keeps the code backward compatible 

## How we teach this

- Create detailed documentation with step-by-step guides
- Update React Native's official documentation to include Swift examples
- Provide sample projects demonstrating real-world use cases


