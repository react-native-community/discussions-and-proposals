---
title: A declarative approach to Xcode projects
author:
- Riccardo Cipolleschi
date: 2023-06-19
---

# RFC0672: A declarative approach to Xcode projects

## Summary

Managing `Xcodeproj` files is error prone and time consuming.

In the past, we had to face different problems with Xcode project files:
- They are large files in user space we have to maintain.
- Once an app is created from the template, we are pushing the same burden on our users.
- They are files with a non standard syntax, for which there is no tooling to work effectively with them (No syntax highlights, for example).
- Those files were burying parts of the build APIs that are needed to build a React Native App.
- Users that are maintaining apps created from the React Native template were having hard time updating their apps, missing some changes that happened in the project file.

The current proposal suggests to move away from manually managed `Xcodeproj` files (more on what this means below) in favor of tools that allows for a more declarative project description. There are different tools in the OSS space that allows to achieve this result and that can generate a project file from that description.

The suggested solution is to adopt [Tuist.io](https://tuist.io) in the App template. This allows to describe the project in Swift, the default language for Apple development.

This tool offers some immediate benefits, depending on the user's category:
- For the React Native team, less maintenance cost. We only need to maintain a file that is ~100 LOC rather than ~1200 LOC.
- For React Native users that are using an app created from the CLI:
    - fewer code in user space (again, 100 LOC rather than 1200).
    - easier way to spot difference when upgrading their iOS project.
- For React Native users of brownfield applications:
    - a clearer description of the public APIs of the build system that are needed to integrate React Native in their Native app.
- For framework developers:
    - a clearer description of the public APIs of the build system that are needed to build a React Native app, and a clearer starting point for the framework.
- For everyone: adding/renaming iOS files in React Native project does not have to happen within Xcode.

Moreover, the project description file is executable Swift code that runs when creating the project. We can use it to run some scripts ahead of time. This provide more flexibility than a pure markup language.

## Basic example

You can have a look at it in action in this PR:

- https://github.com/facebook/react-native/pull/37952

The idea is to describe the project with a declarative approach using Swift. Tuist allows you to load the `Project.swift` file in an instance of Xcode throught the `tuist edit` command. This allows to leverage code completion, type checking and syntax highlight to ensure the correctness of the project. The project description can also be built by Xcode, to verify that it will generate something valid.

### What does it means for the final users

We can separate the usage of the tool in different categories, based on when the tool comes into play:

#### 1. For a user creating a new app from the Template.

Nothing changes.
The user will run:

```bash
npx react-native init <AppName>
```

This will:
- download the template  - already doing that
- run `yarn` - already doing that
- eventually installing Tuist<sup>*</sup> (more on this later) - _new step_
- run `tuist generate` to create the project - _new step_
- run `bundler install` - already doing that
- run `pod install` - already doing that

From the user perspective, the two new steps will be transparent.

#### 2. For a user upgrading a CLI app from a version without Tuist to a version with Tuist (e.g.: from 0.72 to 0.73)

When performing the upgrade, the upgrade-helper will show the following steps:

- Remove the `xcodeproj` file, removing a file that is 1200 LOC
- Add a Swift file, currently ~100 LOC
- add a few lines in the project `.gitignore`

The upgrade helper will also suggest them to install Tuist<sup>*</sup> and run `tuist generate`<sup>**</sup> to create a Xcodeproj (which is ignored by the `.gitignore`) to run their app.

#### 3. For a user upgrading a CLI app from two versions of React Native that uses Tuist (e.g.: from 0.73 to 0.74)

When performing the upgrade, the upgrade-helper will show the following steps:

- Changes in the `Project.swift` file, if any.
- Suggestion to re-run `tuist generate`<sup>**</sup> to create an updated version of the project, if there were changes in the project.

#### 4. For an advanced user (brownfield app maintainer or framework developer)

They can read a ~100 LOC file which is human readable to see if there are changes in the public build API for iOS rather then looking through 1200 LOC.

Just to show some differences to appreciate it, this is how the build settings section looks like for the DEBUG configuration in the Xcodeproj and with a `Project.swift`'s Tuist file:

| Xcodeproj | Swift |
| --- | --- |
| <pre>//&nbsp;Starts&nbsp;at&nbsp;line&nbsp;482<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;13B07F941A680F5B00A75B9A&nbsp;/*&nbsp;Debug&nbsp;*/&nbsp;=&nbsp;{<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;isa&nbsp;=&nbsp;XCBuildConfiguration;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;baseConfigurationReference&nbsp;=&nbsp;3B4392A12AC88292D35C810B&nbsp;/*&nbsp;Pods-HelloWorld.debug.xcconfig&nbsp;*/;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;buildSettings&nbsp;=&nbsp;{<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ASSETCATALOG_COMPILER_APPICON_NAME&nbsp;=&nbsp;AppIcon;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_ENABLE_MODULES&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CURRENT_PROJECT_VERSION&nbsp;=&nbsp;1;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ENABLE_BITCODE&nbsp;=&nbsp;NO;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;INFOPLIST_FILE&nbsp;=&nbsp;HelloWorld/Info.plist;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LD_RUNPATH_SEARCH_PATHS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"@executable_path/Frameworks",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MARKETING_VERSION&nbsp;=&nbsp;1.0;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OTHER_LDFLAGS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-ObjC",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-lc++",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PRODUCT_BUNDLE_IDENTIFIER&nbsp;=&nbsp;"org.reactjs.native.example.$(PRODUCT_NAME:rfc1034identifier)";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PRODUCT_NAME&nbsp;=&nbsp;HelloWorld;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SWIFT_OPTIMIZATION_LEVEL&nbsp;=&nbsp;"-Onone";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SWIFT_VERSION&nbsp;=&nbsp;5.0;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;VERSIONING_SYSTEM&nbsp;=&nbsp;"apple-generic";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;};<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name&nbsp;=&nbsp;Debug;<br/>&nbsp;&nbsp;&nbsp;&nbsp;};<br/>&nbsp;&nbsp;&nbsp;&nbsp;13B07F951A680F5B00A75B9A&nbsp;/*&nbsp;Release&nbsp;*/&nbsp;=&nbsp;{<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//&nbsp;...&nbsp;omitted,&nbsp;this&nbsp;is&nbsp;the&nbsp;Release&nbsp;configuration&nbsp;..<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//&nbsp;these&nbsp;are&nbsp;24&nbsp;lines<br/>&nbsp;&nbsp;&nbsp;&nbsp;};<br/>&nbsp;&nbsp;&nbsp;&nbsp;83CBBA201A601CBA00E9B192&nbsp;/*&nbsp;Debug&nbsp;*/&nbsp;=&nbsp;{<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;isa&nbsp;=&nbsp;XCBuildConfiguration;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;buildSettings&nbsp;=&nbsp;{<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ALWAYS_SEARCH_USER_PATHS&nbsp;=&nbsp;NO;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_ANALYZER_LOCALIZABILITY_NONLOCALIZED&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_CXX_LANGUAGE_STANDARD&nbsp;=&nbsp;"c++17";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_CXX_LIBRARY&nbsp;=&nbsp;"libc++";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_ENABLE_MODULES&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_ENABLE_OBJC_ARC&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_BLOCK_CAPTURE_AUTORELEASING&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_BOOL_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_COMMA&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_CONSTANT_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_DEPRECATED_OBJC_IMPLEMENTATIONS&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_DIRECT_OBJC_ISA_USAGE&nbsp;=&nbsp;YES_ERROR;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_EMPTY_BODY&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_ENUM_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_INFINITE_RECURSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_INT_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_NON_LITERAL_NULL_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_OBJC_IMPLICIT_RETAIN_SELF&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_OBJC_LITERAL_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_OBJC_ROOT_CLASS&nbsp;=&nbsp;YES_ERROR;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_QUOTED_INCLUDE_IN_FRAMEWORK_HEADER&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_RANGE_LOOP_ANALYSIS&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_STRICT_PROTOTYPES&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_SUSPICIOUS_MOVE&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN_UNREACHABLE_CODE&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CLANG_WARN__DUPLICATE_METHOD_MATCH&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"CODE_SIGN_IDENTITY[sdk=iphoneos*]"&nbsp;=&nbsp;"iPhone&nbsp;Developer";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;COPY_PHASE_STRIP&nbsp;=&nbsp;NO;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ENABLE_STRICT_OBJC_MSGSEND&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ENABLE_TESTABILITY&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"EXCLUDED_ARCHS[sdk=iphonesimulator*]"&nbsp;=&nbsp;"";<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_C_LANGUAGE_STANDARD&nbsp;=&nbsp;gnu99;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_DYNAMIC_NO_PIC&nbsp;=&nbsp;NO;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_NO_COMMON_BLOCKS&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_OPTIMIZATION_LEVEL&nbsp;=&nbsp;0;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_PREPROCESSOR_DEFINITIONS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"DEBUG=1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_SYMBOLS_PRIVATE_EXTERN&nbsp;=&nbsp;NO;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_64_TO_32_BIT_CONVERSION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_ABOUT_RETURN_TYPE&nbsp;=&nbsp;YES_ERROR;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_UNDECLARED_SELECTOR&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_UNINITIALIZED_AUTOS&nbsp;=&nbsp;YES_AGGRESSIVE;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_UNUSED_FUNCTION&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;GCC_WARN_UNUSED_VARIABLE&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IPHONEOS_DEPLOYMENT_TARGET&nbsp;=&nbsp;13.4;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LD_RUNPATH_SEARCH_PATHS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;/usr/lib/swift,<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LIBRARY_SEARCH_PATHS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"\"$(SDKROOT)/usr/lib/swift\"",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"\"$(TOOLCHAIN_DIR)/usr/lib/swift/$(PLATFORM_NAME)\"",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"\"$(inherited)\"",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MTL_ENABLE_DEBUG_INFO&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ONLY_ACTIVE_ARCH&nbsp;=&nbsp;YES;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OTHER_CPLUSPLUSFLAGS&nbsp;=&nbsp;(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(OTHER_CFLAGS)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_NO_CONFIG",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_MOBILE=1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_USE_LIBCPP=1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;);<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SDKROOT&nbsp;=&nbsp;iphoneos;<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;};<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name&nbsp;=&nbsp;Debug;<br/>&nbsp;&nbsp;&nbsp;&nbsp;};<br/>//&nbsp;ends&nbsp;at&nbsp;line&nbsp;604<br/><br/></pre>|<pre><br/>settings:&nbsp;.settings(<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;base:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"GCC_C_LANGUAGE_STANDARD"&nbsp;:&nbsp;"gnu99",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"SWIFT_VERSION":&nbsp;"5",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"LD_RUNPATH_SEARCH_PATHS"&nbsp;:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"/usr/lib/swift"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"LIBRARY_SEARCH_PATHS"&nbsp;:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(SDKROOT)/usr/lib/swift",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(TOOLCHAIN_DIR)/usr/lib/swift/$(PLATFORM_NAME)"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"OTHER_CPLUSPLUSFLAGS":&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(OTHER_CFLAGS)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_NO_CONFIG",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_MOBILE=1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_USE_LIBCPP=1"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"OTHER_LD_FLAGS":&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-ObjC",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-lc++",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"CLANG_ENABLE_MODULES":&nbsp;"YES",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"CURRENT_PROJECT_VERSION":&nbsp;"1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"MARKETING_VERSION":&nbsp;"1.0",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"VERSIONING_SYSTEM":&nbsp;"apple-generic"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;configurations:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.debug(name:&nbsp;"Debug",&nbsp;settings:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"ENABLE_BITCODE":&nbsp;"NO",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"GCC_SYMBOLS_PRIVATE_EXTERN":&nbsp;"NO",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"SDKROOT":&nbsp;"iphoneos"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;]),<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.release(name:&nbsp;"Release",&nbsp;settings:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"COPY_PHASE_STRIP":&nbsp;"YES"<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;])<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;defaultSettings:&nbsp;.none</pre>| <pre>settings:&nbsp;.settings(<br/>&nbsp;&nbsp;base:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;"GCC_C_LANGUAGE_STANDARD"&nbsp;:&nbsp;"gnu99",<br/>&nbsp;&nbsp;&nbsp;&nbsp;"SWIFT_VERSION":&nbsp;"5",<br/>&nbsp;&nbsp;&nbsp;&nbsp;"LD_RUNPATH_SEARCH_PATHS"&nbsp;:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"/usr/lib/swift"<br/>&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;"LIBRARY_SEARCH_PATHS"&nbsp;:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(SDKROOT)/usr/lib/swift",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(TOOLCHAIN_DIR)/usr/lib/swift/$(PLATFORM_NAME)"<br/>&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;"OTHER_CPLUSPLUSFLAGS":&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(OTHER_CFLAGS)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_NO_CONFIG",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_MOBILE=1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-DFOLLY_USE_LIBCPP=1"<br/>&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;"OTHER_LD_FLAGS":&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"$(inherited)",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-ObjC",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"-lc++",<br/>&nbsp;&nbsp;&nbsp;&nbsp;],<br/>&nbsp;&nbsp;&nbsp;&nbsp;"CLANG_ENABLE_MODULES":&nbsp;"YES",<br/>&nbsp;&nbsp;&nbsp;&nbsp;"CURRENT_PROJECT_VERSION":&nbsp;"1",<br/>&nbsp;&nbsp;&nbsp;&nbsp;"MARKETING_VERSION":&nbsp;"1.0",<br/>&nbsp;&nbsp;&nbsp;&nbsp;"VERSIONING_SYSTEM":&nbsp;"apple-generic"<br/>&nbsp;&nbsp;],<br/>&nbsp;&nbsp;configurations:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;.debug(name:&nbsp;"Debug",&nbsp;settings:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"ENABLE_BITCODE":&nbsp;"NO",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"GCC_SYMBOLS_PRIVATE_EXTERN":&nbsp;"NO",<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"SDKROOT":&nbsp;"iphoneos"<br/>&nbsp;&nbsp;&nbsp;&nbsp;]),<br/>&nbsp;&nbsp;&nbsp;&nbsp;.release(name:&nbsp;"Release",&nbsp;settings:&nbsp;[<br/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"COPY_PHASE_STRIP":&nbsp;"YES"<br/>&nbsp;&nbsp;&nbsp;&nbsp;])<br/>&nbsp;&nbsp;],<br/>&nbsp;&nbsp;defaultSettings:&nbsp;.none<br/>)</pre> |

## Motivation

Managing Xcodeproj files is error prone and time consuming. They use a proprietary file format to describe the project, using randomly generated UUIDs to refer to projects objects like files, targets, and build configurations.

[This](https://github.com/facebook/react-native/blob/main/packages/react-native/template/ios/HelloWorld.xcodeproj/project.pbxproj) is an example of the Xcodeproj of an app created from the template.

In the past, we had to face different problems with Xcode project files:
- They are large files in user space we have to maintain.
- Once an app is created from the template, we are pushing the same burden on our users.
- They are files with a non standard syntax, for which there is no tooling to work effectively with them (No syntax highlights, for example).
- Those files were burying parts of the build APIs that are needed to build a React Native App.
- Users that are maintaining apps created from the React Native template were having hard time updating their apps, missing some changes that happened in the project file.

The industry has also suffered from managing Xcodeproj since the beginning of iOS development: the pain of managing those files has created various tools in the industry to address this problem. For example, [XCodeGen](https://github.com/yonaskolb/XcodeGen), [Xcake](https://github.com/igor-makarov/xcake/) and [Tuist](https://tuist.io).

Even Apple itself, who owns the standard for the Xcodeproj, started moving away from those projects for libraries and CLI application: for those targets, the suggested approach is to use [Swift Package Manager (SwiftPM)](https://www.swift.org/package-manager/). Notably, SwiftPM **can't** be used to describe iOS Apps project. The supported [products](https://developer.apple.com/documentation/packagedescription/product) currently are:
- Library
- Executable
- Plugin

where the `Executable` product can only be a CLI application, not even MacOS apps.

The current proposal suggests to move away from manually managing `Xcodeproj` files (see the section below to know more about this) in favor of tools that allows for a more declarative project description. There are different tools in the OSS space that allows to achieve that and that can generate a project file from that description.

The suggested solution is to adopt [Tuist.io](https://tuist.io) in the App template. This allows to describe the project in Swift, using the default language for Apple development.

This tool offer some immediate benefits, for different users:
- For the React Native team, less maintenance cost. We only need to maintain a file that is ~100 LOC rather than ~1200 LOC.
    _This is a future proof improvement. Although in recent versions we haven't manipulated the project a lot, we cannot know what the future holds. Apple could always decide that we need some other changes or we can have to manipulate the project file in a future release. I'd personally rather modify 100 lines of legible and typechecked code rather than having to deal with a verbose and error-prone file._
- For React Native users that are using an app created from the CLI:
    - Fewer code in user space (again, 100 LOC rather than 1200).
    - Easier way to spot difference when upgrading their iOS project.
    (For context, this is an example of [changes from 0.67 to 0.68](https://react-native-community.github.io/upgrade-helper/?from=0.67.5&to=0.68.7#RnDiffApp-ios-RnDiffApp.xcodeproj-project.pbxproj), and [this for 0.68 to 0.69](https://react-native-community.github.io/upgrade-helper/?from=0.68.7&to=0.69.11#RnDiffApp-ios-RnDiffApp.xcodeproj-project.pbxproj)
- For React Native users of brownfield applications:
    - a clearer description of the public APIs of the build system that are needed to integrate React Native in their Native app.
- For framework developers:
    - a clearer description of the public APIs of the build system that are needed to build a React Native app and a clearer starting point for the framework.
- For everyone: adding/renaming iOS files in React Native project does not have to happen within Xcode.

For context, [we have been asked](https://github.com/react-native-community/discussions-and-proposals/issues/662) to define the public build APIs in the past.

Moreover, the project description file is executable Swift code that runs when creating the project, so we can use it to run some scripts ahead of time. This provide more flexibility than a pure markup language.

### What do we mean for "Manually Managing Xcodeproj"

To put it simple, **manually managing Xcodeproj** files means to check those files in the vesrioning system and keep them updated.

This creates 2 order of problems:
1. General for any kind of iOS projects
2. Specific for React Native

#### General problems

Having a single, checked-in project file means that, if multiple developers works on the projects, their changes can create conflicts on the Xcodeproj file.

For how Xcode works, every time you modify a settings, add a file to your project or change a script, the project files get modified.

It is common teams that work at the same time on a project, that they have to add new files to implement a features. This generates changes to the Xcodeproject, which results in randomly generated UUID to refer to these new files and there will be conflicts.

For example:
1. Team A adds some files
2. Team B adds some other files.
3. Xcode puts those files, with their UUIDs in the same location in the project description.
4. That generates a git conflict on several lines which seems unrelated that need to be handled manually.
5. It is super easy to make mistakes that results in an invalid project.

Notice that this problem **also** affect React Native. Imagine a team that creates a Native Module or a Native Component in their app: they have to write native iOS code and to do so, they have to manage the Xcodeproj changes.

#### React Native specific problems

There are various React Native specific problems, on top of the one listed above.

1. **Changes in build API are buried into the project and are hard to spot**. As we saw, the Xcodeproj format is verbose and hard to read and it's easy to miss those changes. These makes it hard for brownfield developer and framework developers to apply those changes in their projects, and makes it hard for users not using a framework to update their apps because they can miss those changes.
2. **Repeated information**. The same information is repeated in several parts of the project. For example, Xcode has 2 way to define build settings (Build configuration directly in the Xcode proj and Build configuration specified in the `xcconfig` files. Despite the `xcconfig` files are external files, they are also reported in the Xcodeproj). Usually there are 2 config to handle (Debug and Release). This ends up in having to maintain at least 4 different configurations with the same settings.
3. **Build script are hard to spot and cannot be validated**. Similar to 1, Post Build scripts are plain texts and buried into the verbosity of the project. Several times, people made mistakes forgetting to update those scripts. From an internal perspective, we cannot really validate that those scripts are syntactically valid and correct as they are plain text.
4. **Changes on the file system are not picked up by Xcode**. Renaming a file or adding a file **has** to happen within Xcode. Xcode does not have a mechanism to crawl the file system when it is launched to spot if some files have changed, nor is listening to changes in the file systems. The only way to apply those changes is within Xcode. For example, when migrating from 0.67 to 0.68, several people renamed the `AppDelegate.m` to `AppDelegate.mm` outside the Apple IDE, ending up with a broken project. The tools solve this problem by listing source folders or path so that a project is created using the content on the disk and not a static list of files predefined in a file.

## Detailed design

There are different ways in which we can adopt a Tuist within the template. I'll start listing them from what I think is the ideal solution to the least ideal.

### Fully Managed flow:

1. Update the template using Tuist (landing https://github.com/facebook/react-native/pull/37952). This PR already contains a functioning project which passed all the iOS tests in CI.
1. Update the React Native dependencies to depend on a Tuist NPM package that ships the executable. This allow us to avoid installing the executable on the user machine
    **Dependency:** We need the Tuist.io team to publish such package. This suggestion came from [@pepicrft](https://github.com/pepicrft) who is one of the core maintainer of Tuist.
1. Update the CLI and instruct it to swap the `HelloWorld` strings in the `template/ios/Package.swift` to the `AppName` chosen by the user.
1. Update the CLI to run `tuist generate` to create the project
1. Update the CLI to run `tuist generate` on `yarn ios` or `npm run ios` in case the `Project.swift` has changed or there is no `Xcodeproj` file.
1. Remove the old `Xcodeproj` file and set the right lines in the `.gitignore` settings.
1. Update the React Native scripts in a way that runs the `tuist generate` command automatically when the user runs `yarn ios`<sup>**</sup>

<sup>&#42;</sup>: In the **"What does it means for the final users"** section, we marked `install tuist` with an <sup>*</sup>. If we manage to collaborate with the Tuist team, we can save the installation step, having an NPM package who provide an executable for users downloading react native. This has also the benefit of making sure that each React Native release will come with a compatible Tuist version.

<sup>**</sup>: As part of the H2 work I'm going to carry on, I'll be working on optimize the way we run our apps from the command line. I'd like to simplify the way in which we install dependencies and build iOS using a single command, instead of having to run `pod install` with multiple flags. If this happens, that single commands will also take care of regenerating the project under the hoods.

### Half Managed flow

1. Update the template using Tuist (landing https://github.com/facebook/react-native/pull/37952). This PR already contains a functioning project which passed all the iOS tests in CI.
1. Update the CLI to check whether there is a version of Tuist installed in the system. If not, it installs it and leverages `tuistenv` and the project `.tuist-version` file to set the proper tuist version that is compatible with the project.
1. Update the CLI and instruct it to swap the `HelloWorld` strings in the `template/ios/Package.swift` to the `AppName` chosen by the user.
1. Update the CLI to run `tuist generate` to create the project
1. Update the CLI to run `tuist generate` on `yarn ios` or `npm run ios` in case the `Project.swift` has changed or there is no `Xcodeproj` file.
1. Remove the old `Xcodeproj` file and set the right lines in the `.gitignore` settings.
1. Update the React Native scripts in a way that runs the `tuist generate` command automatically when the user runs `yarn ios`<sup>**</sup>

### Not Managed flow

1. Update the template using Tuist (landing https://github.com/facebook/react-native/pull/37952) This PR already contains a functioning project which passed all the iOS tests in CI.
2. We do not modify the CLI at all and we do not ask our users to install or use Tuist at all. We don't even ship the Project.swift to the final user. Instead, we update the release pipeline so that it run tuist while releasing a new version of React Native.

With this approach, we minimize changes and users who want to keep working with Xcodeproj won't feel any difference.
We will have a clearer description of the build API in the React Native repo and we will use it to generate the Xcodeproj for the user.

This approach has the specific drawback of adding some more files in the user space. So, when they have to migrate, they will need to add some other files to their project and they will have to deal with upgrading the Xcodeproj file rather than just deleting it.

## Drawbacks that affect all the proposed solutions

- **Prioritization:** I think that the only reason NOT to do this is for prioritization and time. Implementing this system will take some time which we can devote to other, maybe more impactful, activities. However, this will make the management of the project easier for the team in the future.
- **Dependency:** React native will start to depend on an additional tool, on top of the other tools it already depends on (Gradle, Cocoapods, ...). If Tuist stop being maintained, we will have to figure out how to move forward. However, Tuist is currently supported by several companies and we have maintainers of Tuist which are also helping maintaining React Native, so this risk is mitigated by having a direct contact with the Tuist.io team. Also, big companies are using it, which make it more stable and less likely to go unmaintained. For example, [American Express](https://github.com/sponsors/tuist) and [Sky](https://tuist.io/ scroll down to _testimonies_)
- **Workflow changes:** there is an extra step that user has to run when modifying the project: after every change in the `Project.swift`, users have to run `tuist generate`. This is a minor issue as we do not expect for the user to manually update the `Project.swift` file at all, and we can bake most of the Tuist invocations in the CLI.

## Alternatives

### [Swift PM](https://www.swift.org/package-manager/)

This tool allows to describe packages using Swift. Tuist took inspiration from the Swift PM's approach.

This is not really an alternative as SwiftPM does not support the creation of iOS or MacOS apps yet. We may be considering it in the future if something changes, but so far that does not seems likely.

### [XCodeGen](https://github.com/yonaskolb/XcodeGen)

This tool allows to describe the Xcodeproj using a YAML file.

If we go for this approach, the main drawbacks are:

- a new language to teach to our iOS users. YAML could be more familiar to web developers, but we are talking about a tool to manage Xcodeproj files, so I think it is better to use a tool that is closer to the native platform.
- more verbose project description than Tuist.
- no code completion: Tuist scripts can be loaded inside Xcode and we can leverage the Xcode standard tooling to check types and syntax for the project.
- Xcodegen uses a markup language. This makes it less flexible than Tuist which can run code at project-generation time.
- Xcodegen is harder to extend, as we are limited by the markup entities defined by the Xcodegen version we are currently using.
- Single person project. Xcodegen lives in a public repository of a github user, so they can decide to delete the repo or to drop its support.

### [Xcake](https://github.com/igor-makarov/xcake/)

This tool allows to describe a project in Ruby.

If we go for this approach, the drawbacks are:

- a new language to teach to our users.
- no type checking due to the dynamic nature of Ruby
- one of the oldest tool: it has been unmaintained for a few years before some other OSS contributor took ownership of it. It could go unmaintained again.
- Single person project. Xcake lives in a public repository of a github user, so they can decide to delete the repo or to drop its support.

## Adoption strategy

- Implement all the changes before 0.73.
- Communicate with a BlogPost the change, explaining the implications for new apps and updates.

## How we teach this

- Add the requirement to the website
- Blogposts
- Automating the workflow as much as possible.
