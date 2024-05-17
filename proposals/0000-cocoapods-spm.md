---
title: Declare swift package manager dependencies in react-native libaries podspec
author:
- Miklós Fazekas
date: 2024-05-17
---

# RFC0000: declare swift package manager dependencies in react-native libaries podspec

## Summary

Currently there is no straightforward way for react-native libraries to consume packages from swift package manager. We address this by adding a ruby method that allows library podspec to declare SPM depencendy and we also add the neccesary cocoapods postinstall hooks to make the chane to the xcode files.

## Basic example

```
if const_defined?(:ReactNativePodsUtils) && ReactNativePodsUtils.respond_to?(:spm_dependency) 
   ReactNativePodsUtils.spm_dependency(s,  
     url: 'https://github.com/apple/swift-atomics.git', 
     requirement: {kind: 'upToNextMajorVersion', minimumVersion: '1.1.0'}, 
     products: ['Atomics'] 
   ) 
 else 
   raise "This ReactNative doesn't support SPM dependency" 
 end 
```

## Motivation

Swift Package Manager is the Apple supported standard for iOS packages. While many packages are published to both CocoaPods and Swift Package Manager, many packages are Swift Package Manager exclusive. This change enables react-native native modules to consume Swift Package Manager packages.

## Detailed design

ReactNative cocoapod utils implements `ReactNativePodsUtils.spm_dependency(podspec, spmspec)` stores the spmspec for the podspec. Then later at CocoaPod postinstall hook, we modify:
- .xcworkspace to add the swift pacakge to `Package Dependencies`
- in the Pods project it adds the swift package to `Link with Libraries` build phase 

See https://github.com/mfazekas/rn-spm-rfc-poc for a POC. https://github.com/mfazekas/rn-spm-rfc-poc/blob/main/example/patches/react-native%2B0.74.1.patch contains the changes in react-native.

## Drawbacks

- Feature can be implemented in .podspec, but user's are required to call the module post_install, which is not convinient. https://github.com/maplibre/maplibre-react-native/blob/0ea35c4c3d0dd1a395423f322342128b250d8192/maplibre-react-native.podspec#L26
- There are some bugs/issues regarding Swift Package Manager, like `-Objc` flag used by default config in rn (causes duplicate symbols with Swift Package manager)[https://forums.swift.org/t/objc-flag-causes-duplicate-symbols-with-swift-packages/27926]. Those will be considered as issue in react-native.      

## Alternatives

- Ideally this functionality belongs to CocoaPods, but the (PR addressing this)[https://github.com/CocoaPods/CocoaPods/issues/11942] has not moved forward, in the last year. (FWIW Once cocoapods implement this then RN implementation could be a trivial)
- Feature could be also implemented as CocoaPods plugin (drawback is that customer has to install it)
- We could also adopt other solution than cocoapods that supports SPM, like SPM itself, but the scope of that solution is huge, while this is a simple change.

See also:
- (Add support for Swift Package Manager (SPM) as the dependency manager for iOS)[https://github.com/react-native-community/discussions-and-proposals/issues/587]
- (Update 0004-cocoapods-support-improvements.md)[https://github.com/react-native-community/discussions-and-proposals/pull/439]

## Adoption strategy

Once we have this functionlatiy libraries can use it, or raise some error if it's not there.

```ruby
 if const_defined?(:ReactNativePodsUtils) && ReactNativePodsUtils.respond_to?(:spm_dependency) 
   ReactNativePodsUtils.spm_dependency(s,  ...)
  else
    raise "This React Native doesn't support spm depnedencies, please upgrade"  
  end
```

## How we teach this

?

## Unresolved questions

- Not sure how to handle the (-ObjC flag + SPM linker error issue)[https://forums.swift.org/t/objc-flag-causes-duplicate-symbols-with-swift-packages/27926] 



