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

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

## Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React Native
- integration of this feature with other existing and planned features
- cost of migrating existing React Native applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? Why did you select your approach?

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

## How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers?

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?
