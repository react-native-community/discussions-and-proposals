---
title: Adding support for Swift / Moving away from Obj-C on Apple platforms
author:
- Parsa Nasirimehr
date: 2023-12-14
---

# RFC0000: Adding support for Swift / Moving away from Obj-C on Apple platforms

## Summary

React Native's base code on Apple's platfroms has been written in a mix of Obj-C, Obj-C++ and C++. This proposal discuses what path we might have to migrate the codebase to using just C++ and Swift, Apple's current standard language for developing apps on their platforms, and what areas we need to be to mindful of (including tooling, various architectures that would need to be supported, documentation updates and the impact on third party libraries)

## Motivation

The motivation for this change is threefold:
- As of Swift 5.9, Swift is now capable of direct interop with C++, Which was one of the primary reasons Obj-C and Obj-C++ were still required within the repository. While the development [is still ongoing](https://www.swift.org/documentation/cxx-interop/status/) and there are [constraints that still exist](https://github.com/apple/swift/issues/66159), Apple's end goal is to reach the needed API parity to work seemlesssly between Swift and C++
- Obj-C is relatively hard to write and maintain for, and it is reflected by the [community](https://github.com/react-native-community/discussions-and-proposals/issues/104). Having a modern language that is more readable, more performant(in some cases) and [safer from certain class of errors](https://developer.apple.com/swift/#safety) means that users and library maintainers alike can produce better quality code and take advantage of the underlying platforms easier if they need it
- As Swift evolves and becomes more feature rich, there is a possibility that Apple will begin to remove support for Obj-C in it's toolchain. The documentation for the various APIs that Apple provides may also be moved to simply having Swift as the supported language.


## Detailed design

### HELP WANTED: I am unaware of a lot of the internals of RN, so if anyone more aware than me can contribute, it is appreciated.

The general idea is that the code we have for Apple platforms should be consistent of three primary parts by the end of this migration:
- The Swift API layer that is in charge of the platform specific operations that can not be shared across other platforms (the View Delegate, the Scene Delegate, anything that has to do with Apple specific frameworks like UIKit)
- The Shared C++ code that is platform agnostic and used by all React Native target platforms (Yoga, Hermes, the event handlers, the bridge (if applicable), etc)
- The optional intermidery C++ code, with Swift specific annotations for certain APIs that help Swift interact with C++ in a more efficient and correct manner (To learn more, please refer to [Swift's documentation of interop with C++](https://www.swift.org/documentation/cxx-interop/#exposing-swift-apis-to-c) under "Customizing How C++ Maps to Swift")

The third step is marked as optional. The idea is that the Swift compiler makes certain assumptions when interacting directly with the C++ modules that may not necessarily be correct (C++ types being imported as value types by default, for instance). But we can override those behaviours by explicitly marking certain sections with the annotations provided to us by `<swift/bridging>` (Marking things to be imported as reference types, Mapping getters and setters, etc)

Much like our current setup with Obj-C, The Apple platform section of any new project template created will simply expose the Swift files for developers to extend and add functionalities to. The React Native APIs can be simply imported via a swift import and consumed by the Swift parts, giving us control over our API surface and hiding away any specific pieces that will not be needed on a day to day basis. Example:

```swift
import React

@main
class AppDelegate: RCTAppDelegate {
    override func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        self.moduleName = "HelloWorld"
        // You can add your custom initial props in the dictionary below.
        // They will be passed down to the ViewController used by React Native.
        self.initialProps = [:]

        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }

    override func sourceURL(for bridge: RCTBridge!) -> URL! {
        return self.getBundleURL()
    }

    func getBundleURL() -> URL {
        #if DEBUG
        return RCTBundleURLProvider.sharedSettings().jsBundleURL(forBundleRoot: "index")
        #else
        return Bundle.main.url(forResource: "main", withExtension: "jsbundle")!
        #endif
    }

    /// Native bits of functionality that users may extend and add on their own
}
```

## Drawbacks

The biggest drawback can be boiled down to certain key elements:
- It is a massive undertaking, that will likely take years to fully complete
- The Swift API is not yet fully compatible with everything we will need on the C++ side of things
- Libraries will need to update their documentation and probably some changes to their imports as well (ideally, the API should not change, unless it is related to the Platform specific side of things and the Swift implementation simplifies the existing API for them to consume) and as [previously established](https://github.com/react-native-community/discussions-and-proposals/issues/671) not all libraries adapt and maintainers may not update
- Users will need to upgrade their templates and the libraries they use to Switch over

## Alternatives

### Leaving things as is
The overhead of this option is much much lower in the short term, and while it creates a bad user experience for the smaller subsection of the user base that want to be able to do more (and library maintainers who would probably benefit from this new found access), the cost is undeniably lower. The problem is that eventually, Apple will either push for this change directly, or will make it very hard to continue, so unless we take initiative now, We will be forced to it down the road, and in the mean time React Native will have to resort to more and more custom solutions to deal with any APIs that might become unavailable

## Adoption strategy

The core idea is to still be able to deliver feature and changes that users and library maintainers have been asking for while doing the migration. One strategy that comes to mind for achieving this result is as follows:
- (If needed) Isolating the OSS bits that are user facing from the internals as much as possible (Like AppDelegate and the tests)
- (If needed) Updating Cocopods infra
- (If needed) Communicating with third party libraries to adjust
- (If needed) Rolling out a new version
- Migrating the OSS parts from Obj-C -> Swift
- Updating CLI to support the new files and structures
- Updating React Native Helper to support people trying to migrate their templates from Obj-C to Swift
- Updating the docs
- Communicating with library maintainers if help is needed updating their documentation to support the setup for both Swift and Obj-C for older users
- Rolling out a new version
- Moving as much shared logic as possible to C++ and away from Obj-C, separating them as distinctly as possible while adding new features and capabilities and continuing to roll out new versions (**Users should not be affected, and ideally the Library maintainers should not be affected either**)
- Beginning the transition from Obj-C to Swift from the tests in the internals (**No effect on users or library maintainers**)
- Moving the rest of the modules one at a time from Obj-C to Swift (**Library maintainers may need to update, if they wish to take advantage of the newer APIs or if we break change an APIs signiture, but ideally it should only be in the direction of making it easier to maintain libraries and better APIs**)
- Updating Cocopods infra
- ? (Please add any more steps that come to mind, at the end or anywhere else in the list)

## How we teach this

This should be a pretty welcome step for the user base (and even the maintainers, if done correctly and without too many breaking changes). The docs for creating Native module will need to be updated to add support for Swift, and we can simply mention the change in React Native EU or similar conferences.

The idea is to make it very un-noticable to the vast majority of the user base who simply want to use the React side for creating their mobile applications, and the part of the community that is capable of creating native modules should simply feel like it is just another native Apple app that they can extend using whatever it is that they need (perhaps adding a watchOS target support for their app, or using CoreML or AppClips or making a SwiftUI based widget)

It should be presented as React Native keeping up with changes and lowering the barrier of entry for all devs and that we are simply making part of the work that is duplicated across platforms easy to reuse, using the same tools that our industry is using and without sacrificing the experience.

## Unresolved questions

- What other parts of the infra need to be changed for this transition to work?
- How will this affect other targets such as React Native MacOS, React Native visionOS, etc?
- Is our current separation of internals from the OSS template enough that we can simply change the template directly without any of the first four steps needed? (I managed to get a [test repo](https://github.com/TheRogue76/React-Native-Template-With-Swift) to work as a proof of concept with these changes, but i am not sure if that means that the app will work for all situations (new arch, bridgless, etc). I also submitted [a PR](https://github.com/facebook/react-native/pull/41896) to test out the current tests we had and those passed, but that might not be enough). And what about Expo? Will it continue to function if this change is done to the base template, or would it interfere with any of the work done on their side
- Can we create a separation between the base template of the app and rest of the internals in a way that would allow us to switch over the template without breaking the everything in the process?

