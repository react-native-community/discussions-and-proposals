---
title: CocoaPods Support Improvements
author:
- Orta Therox
- Eloy Durán
date: 2018-08-15
---

# RFC0003: CocoaPods Support Improvements

## Summary

Simplify CocoaPods support in React Native, by reducing the differences between how CocoaPods maps React Native into the existing set of Xcode projects.

## Basic example

CocoaPods support in React Native is a tad tricky, because it’s trying to map a complex set of Xcode projects into a single dependency. The Podspec for React Native does a lot of work under the hood at ~300 LOC. This is, in part, because we use a CocoaPods abstraction called subspecs to consolidate out the React Native library into one single library.

If we split the `React.podspec` into many podspecs, one per Xcode project. Roughly ~10ish Podspecs per React Native release.  These would match up one-to-one to the Xcode projects so that when CocoaPods support is broken in React Native, an error report would make it obvious which part of the project is broken.

This means we can also move React Native and we can get CocoaPods support back in CI. This route allows CocoaPods support to be tested without network access (all the required information would be inside the same)

The CocoaPods React dependency would then depend on React-ART, and React-RCTSettings etc.

## Motivation

For teams adding React Native to existing apps CocoaPods support let's RN be treated as a dependency like all of the other ones. CocoaPods is built with a lot of escape valves for consumers to fix issues at runtime (you see people passing around `post_install` [fixes](https://github.com/facebook/react-native/issues/20182#issuecomment-409131862) for example, and I built a [CocoaPods plugin](https://github.com/orta/cocoapods-fix-react-native) to try consolidate them all)

I think one of the a key reasons for the drift between the CocoaPods support and the Xcode projects is because they're not mapped together.

## Detailed design

```sh
rm React.podspec

touch React/React.podspec
pod spec lint React/React.podspec

touch Libraries/ART/React-ART.podspec
touch RNTester/React-RNTester.podspec
touch Libraries/Blob/React-RCTBlob.podspec
# etc
```

Implementing that is reasonably straight forwards. Then we have two ideas about how the Podspecs can be distributed.

- Podspecs are shipped to CocoaPods trunk when a release is deployed. This is the simplest for library consumers, but a bit more taxing on release deployment. It must have been discussed before, but they were classed as being deprecated. I'm open for this RFC to be about bringing that back.

- Or... The React Native repo is turned into a Podspecs Repo. This would mean making a Specs folder in the root of the react-native repo, and putting in the CocoaPods Podspecs. People using CocoaPods would include React Native at the top of their Podfiles like so:

```ruby
source 'https://github.com/facebook/react-native.git'
source 'https://github.com/CocoaPods/Specs.git'

pod "React"
pod "React-Devtools"
```

The changes would convert the React Native repo to act like a specs repo:

```
$ ls .

[... as it is today]
Specs
[... as it is today]

$ tree Specs

React
 ├── 0.57.0
 │    └── React.podspec.json
 └── 0.57.1
      └── React.podspec.json
React-ART
 ├── 0.57.0
      └── React-ART.podspec.json
 └── 0.57.1
      └── ...
React-RCTSettings
 ├── 0.57.0
 └── 0.57.1
Yoga
 ├── 0.45.0-rc.2.React
 └── 0.48.4.React
 [.. for all the specs]
```

## Drawbacks

Why should we _not_ do this? Please consider:

- This adds more files for CocoaPods support in the React Native repo, but it puts each one next to the Xcodeproj it represents.
- The "React Native repo as Specs repo" adds a folder called "Specs" in the root, which is confusing for JS/Android folk. We can add support into CocoaPods to recognize "Podspecs" instead also.
- If you were already separating your JS repo from your app code, then you would need to change your Podfile. But this means you don't have to host these Podspecs yourself (or [use Artsy's](https://github.com/artsy/specs))


## Alternatives

- CocoaPods support could be handled by [shipping binaries](https://github.com/react-native-community/discussions-and-proposals/issues/15) of React Native to clients. Personally, I prefer to be working with the source code for ease of forkability and debugging. 

## Adoption strategy

Shouldn't be major, a paragraph or two in the blog post for the version that first supports it and an update to the docs page for integrating with an existing app.

## How we teach this

Shouldn't be an issue here.

## Unresolved questions

The biggest call is:

- Submitting the Podspecs to CocoaPods on a deploy, or
- Turning the "React Native repo into a Spec repo"

If anyone can talk to the original reasons for deprecating trunk support? then I can give a better opinion about which makes the most sense. The first one is simpler for everyone, but in 2016 the trunk support was [deprecated](http://cocoapods.org/pods/React). 
