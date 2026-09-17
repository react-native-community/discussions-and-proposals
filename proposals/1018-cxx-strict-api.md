---
title: Reducing React Native’s C/C++ Public API
author:
- Jakub Piasecki
- Dawid Małecki
date: 19.08.2026
---

# RFC0000: Reducing React Native’s C/C++ Public API

## Summary

Any change to the React Native public API header or symbol might become a breaking change for the apps and libraries that build against it. Currently, there is no distinction between public and private APIs, thus, every symbol is considered public. This problem was mitigated for the JS and Android parts of the framework. We recently introduced the C++ public API snapshot as the first step to covering the remaining part of the React Native API.

The RFC proposes reducing the public API surface by classifying modules into three visibility tiers — public, for frameworks, and private — meaning their symbols are stable across releases, stable within a minor version, or unstable, respectively.

## Motivation

Following the work on the stable JS/Typescript and Java/Kotlin APIs, we are continuing to stabilize React Native by tackling the C++ public surface. Having a clear public API snapshot helps us inspect changes to symbols used by consumers and reduce the number of breaking changes or block them for non-breaking releases.

The current baseline exposes nearly all of ReactCommon, which consists largely of internal implementation details that were never intended for consumer use. This overexposure resulted in problems that have been affecting React Native for the past few years:
- Ecosystem fragility - third-party libraries accidentally depend on unstable APIs, leading to unexpected breaking changes between releases.
- Slowed core development - React Native core team is restricted from changing internal logic since it might affect downstream consumers.
- Lack of information - there is no source of truth for what is considered safe to build against.

### Benefits

Implementing the three-tier visibility system comes with several benefits for the core team and the community:
1. Reducing the number of breaking changes.
2. Catching and preventing unintended public API changes.
3. Calculating the blast radius of a proposed change against a meaningful surface.
4. Establishing a clear, documented distinction between API intended for general public use, framework-level extensions, and private implementation details.


## Detailed design

The snapshot of the public surface is produced with respect to the API view and the variant. The views are `ReactCommon`, `ReactAndroid`, and `ReactApple` generated with `debug`, `release`, and `newarch` variants. For `ReactAndroid` and `ReactApple`, codegen runs beforehand so that TurboModule component headers are included.

The reduction of the Stable API will be conducted per target. Each target is classified as either:
1. Public - considered stable between versions and safe to build against. Any change here will be considered breaking and properly communicated.
2. For frameworks - considered stable within the minor version. Usage of these symbols will result in a suppressible warning.
3. Private - not stable, can change at any time. Usage of these symbols will result in a suppressible error.

Considering the OSS usage and comments from our partners, we propose the following list of **private** targets:

- cxxreact:module
- react/renderer/components/progressbar:androidprogressbar
- react/renderer/components/switch:androidswitch
- react/renderer/components/inputaccessory:inputaccessory
- react/renderer/components/legacyviewmanagerinterop:legacyviewmanagerinterop
- react/renderer/components/safeareaview:safeareaview
- react/renderer/components/unimplementedview:unimplementedview
- react/renderer/components/virtualview:virtualview
- react/performance/timeline:timeline
- cxxreact:bridge
- react/nativemodule/idlecallbacks:idlecallbacks
- react/renderer/leakchecker:leakchecker
- react/renderer/componentregistry/native:native
- react/performance/cdpmetrics:cdpmetrics
- react/renderer/observers/intersection:intersection
- jsiexecutor:jsiexecutor
- react/nativemodule/dom:dom
- react/renderer/observers/mutation:mutation
- react/nativemodule/intersectionobserver:intersectionobserver
- react/nativemodule/mutationobserver:mutationobserver
- devtoolsruntimesettings:devtoolsruntimesettings
- react/nativemodule/webperformance:webperformance
- react/nativemodule/cputime:cputime
- react/renderer/element:element
- react/nativemodule/devtoolsruntimesettings:devtoolsruntimesettings
- react/renderer/observers/events:events
- reactperflogger:reactperflogger
- hermes/executor:executor
- react/renderer/uimanager/consistency:consistency
- react/nativemodule/microtasks:microtasks
- react/nativemodule/viewtransition:viewtransition
- react/networking:networking
- react/renderer/consistency:consistency
- react/renderer/viewtransition:viewtransition
- react/nativemodule/defaults:defaults
- jsinspector-modern/network:jsinspector_network
- hermes/inspector-modern:chrome
- react/renderer/dom:dom
- oscompat:oscompat
- react/renderer/observers/resize:resize
- react/nativemodule/resizeobserver:resizeobserver
- react/nativemodule/featureflags:featureflags
- react/renderer/components/switch:iosswitch
- react/runtime/nativeviewconfig:nativeviewconfig

Consequently, we suggest the following targets to be used by frameworks only:

- react/renderer/runtimescheduler:runtimescheduler
- react/renderer/textlayoutmanager:textlayoutmanager
- react/renderer/telemetry:telemetry
- react/renderer/animated:animated
- react/renderer/bridging:bridging
- jsinspector-modern:jsinspector
- jsinspector-modern/tracing:jsinspector_tracing
- jsinspector-modern/tracing:traceevent
- jsinspector-modern/cdp:jsinspector_cdp
- jsinspector-modern/tracing:profile
- jsitooling:jsitooling
- jserrorhandler:jserrorhandler
- react/renderer/mounting:mounting
- cxxreact:tracesection
- react/renderer/components/modal:modal
- react/renderer/components/text:text
- react/renderer/components/textinput:textinput
- react/renderer/components/root:root
- react/renderer/components/scrollview:scrollview
- react/renderer/animations:animations
- react/renderer/animationbackend:animationbackend
- react/renderer/attributedstring:attributedstring
- react/renderer/uimanager:uimanager
- react/runtime:runtime
- react/runtime/hermes:hermes
- cxxreact:messagequeuethread
- react/renderer/scheduler:scheduler
- jsc:JSCRuntime

Remaining targets will be considered public.

For each public module we introduce one **umbrella header**, named after the module that establishes a single public entry point. It re-includes only the headers that make up the module’s intended public API. Implementation details are excluded and become internal to the build unit.

All umbrellas will share a single `React/` include namespace. There is no central folder that holds them; instead each module maps its own umbrella’s search path to `React/<UmbrellaName>.h`, and the build system aggregates them. Namely, every module that adopts the pattern must add the mapping for its umbrella.

1. Adding an umbrella file.

For a module with public headers we add an umbrella file `<module/React/<Module>.h>` that re-includes `public` headers only with `RN_UMBRELLA_INCLUDE` scoped around them:

```cpp
#pragma once

#define RN_UMBRELLA_INCLUDE
#include <.../PublicHeaderA.h>
#include <.../PublicHeaderB.h>
// for-frameworks and private headers are intentionally NOT included here
#undef RN_UMBRELLA_INCLUDE
```

2. Add guards to headers.

Include `UmbrellaGuard.h` in public headers:

```cpp
#pragma once
#include <react/cxxstableapi/UmbrellaGuard.h>
// ... public declarations ...
```

Include `FrameworksGuard.h` in headers selected for the frameworks usage:

```cpp
#pragma once
#include <react/cxxstableapi/FrameworksGuard.h>
// ... framework-only declarations ...
```

Include stricter `PrivateGuard.h` in private headers (also for headers in private targets):

```cpp
#pragma once
#include <react/cxxstableapi/PrivateGuard.h>
// ... implementation-detail declarations ...
```

All guards are defined as follows:

```cpp
// react/cxxstableapi/UmbrellaGuard.h
#if defined(RN_STRICT_API) && !defined(RN_UMBRELLA_CONTEXT) && !defined(RN_BUILDING)
#error "Do not include this header directly. Include the module umbrella <React/<Module>.h> instead."
#endif

// react/cxxstableapi/FrameworksGuard.h
#if  defined(RN_STRICT_API) && \
     !defined(RN_ALLOW_FRAMEWORKS) && \
     !defined(RN_UMBRELLA_CONTEXT) && \
     !defined(RN_BUILDING)
#  if defined(__GNUC__) || defined(__clang__)
#    pragma GCC warning "RuntimeScheduler.h is not part of the public API of React Native."
#  elif defined(_MSC_VER)
#    pragma message("RuntimeScheduler.h is not part of the public API of React Native")
#  endif
#endif

// react/cxxstableapi/PrivateGuard.h
#if defined(RN_STRICT_API) && \
    !defined(RN_BUILDING)
#error "Private React Native header. Include the module umbrella <React/<Module>.h> instead."
#endif
```

Guards consist of 3 flags defining the inclusion context of the header:

| Macro | Set by | Meaning |
| RN_UMBRELLA_INCLUDE | The umbrella header itself | The include is coming through the umbrella |
| RN_BUILDING | The module’s own building target | The include is from the module’s own sources (e.g. its .cpp files) |
| RN_STRICT_API | The consumer (opt-in) | Turn enforcement on to get a hard error on direct includes |

The `error` only fires when enforcement is turned on and the include did not come through the umbrella and it is not a module building itself, so existing consumers and the OSS builds are unaffected until they opt in. For general use, the `RN_STRICT_API` should be set to ensure only the public API is used.

3. Update build configs.

Map the umbrella to `React/<Module>.h` in build files:
- CMake: expose the module’s `React/` subdirectory as a public include dir so `<React/<UmbrellaName>.h>` resolves for dependents.
- CocoaPods: move the umbrella from the module’s pod via an `Umbrella` subspec with `header_dir = “React”`.

The `RN_BUILDING` flag should also be defined for the module’s own target.

## Basic example

1. Defining umbrella header.

We start with the umbrella header for the `RuntimeScheduler` that might look like this:

```cpp
// react/renderer/runtimescheduler/React/RuntimeScheduler.h
#pragma once

// Mark that subsequent module headers are pulled in through the umbrella, so the
// per-header guard allows them.
#define RN_UMBRELLA_INCLUDE

#include <react/renderer/runtimescheduler/RuntimeScheduler.h>
#include <react/renderer/runtimescheduler/RuntimeSchedulerCallInvoker.h>
#include <react/renderer/runtimescheduler/RuntimeSchedulerEventTimingDelegate.h>
#include <react/renderer/runtimescheduler/RuntimeSchedulerIntersectionObserverDelegate.h>
#include <react/renderer/runtimescheduler/SchedulerPriorityUtils.h>
#include <react/renderer/runtimescheduler/Task.h>
#include <react/renderer/runtimescheduler/primitives.h>
// RuntimeScheduler_Modern.h / RuntimeScheduler_Legacy.h are deliberately NOT included.

#undef RN_UMBRELLA_INCLUDE
```

Here we do not include `RuntimeScheduler_Modern.h` and `RuntimeScheduler_Legacy.h` forks to show the difference between the public and private headers from the umbrella perspective.

2. Adding inclusion guards.

Next, we add three different guards for public, private, and for frameworks headers. Starting from the public ones, the guard enforces umbrella-only inclusions:

```cpp
// react/renderer/runtimescheduler/RuntimeScheduler.h  — public
#pragma once
#include <react/cxxstableapi/UmbrellaGuard.h>

#include <react/renderer/runtimescheduler/Task.h>
// ... RuntimeScheduler declaration ...
```

For private headers, we add a guard that accepts internal includes only:

```cpp
// react/renderer/runtimescheduler/RuntimeScheduler_Modern.h  — private fork
#pragma once
#include <react/cxxstableapi/PrivateGuard.h>

// ... implementation-detail declaration ...
```

Finally, we guard the **for frameworks** headers, but here a direct include produces a warning rather than a hard error:

```cpp
// react/renderer/runtimescheduler/RuntimeSchedulerBinding.h  — for frameworks
#pragma once
#include <react/cxxstableapi/FrameworksGuard.h>

#include <jsi/jsi.h>
// ... RuntimeSchedulerBinding declaration ...
```

Frameworks should set the `RN_FRAMEWORK_ACCESS` flag to use symbols from the “for frameworks” headers.

3. Update CMakeLists.txt

Add `react_cxxstableapi` to the module's existing `target_link_libraries` list so the
guard header is a declared dependency of the Android/OSS CMake build:

```cpp
target_link_libraries(react_performance_timeline
        jsinspector
        jsinspector_tracing
        reactperflogger
        react_cxxstableapi
        react_featureflags
        react_timing
        folly_runtime)
```

4. Update the podspec file

Similarly, the podspec configuration also needs to expose the umbrella header inside the React namespace.

```cpp
Pod::Spec.new do |s|
  s.name                   = "React-runtimescheduler"
  ...
  s.exclude_files          = ["tests", "React"]

  ...

  s.subspec "Umbrella" do |ss|
    ss.source_files        = "React/*.h"
    ss.header_dir          = "React"
    ss.header_mappings_dir = "React"
  end
end
```
5. Consumers migrate to new umbrella includes.

Internal React Native code keeps using the fine-grained includes:

```cpp
#include <react/renderer/runtimescheduler/RuntimeScheduler.h>
```

External users should rely only on the umbrella headers:

```cpp
#include <React/RuntimeScheduler.h>
```

6. Regenerate C++ API snapshots.

The last step is to regenerate the C++ API snapshots using `yarn cxx-api-build` and commit the updated `.api` files.

### Tooling

The tool built for generating the C++ API snapshots was built with the assumption that every reachable header is public. As part of this effort, this tool will need to be updated to only consider header files reachable via the public umbrella entry points.

We expect there may be some inherently private headers inside the public modules as well (for example, forked RuntimeScheduler implementations). To make finding cases like that easier, a tool to build a DAG for the header dependencies could prove useful - the leaves in such DAG are candidates for such cases.


## Drawbacks

Not doing the work:

- The public C++ surface contains all headers, and every change will be `breaking`. We cannot guarantee stability without defining parts of the API that should be private and open to modifications.

Drawbacks of this approach:

- Public API consumers will have to migrate to includes from the umbrella file.
- The guard is only enforceable when consumers opt-in. Nothing prevents direct includes in OSS.

## Alternatives

The alternative approach would be an explicit division of headers into private and public and enforcement at a compiler level by not exposing private headers on header search paths. This, however, comes with its own drawbacks:

- Granular adoption is not possible - the private headers would not be accessible at all for consumers, making it hard for third-party libraries to keep backwards compatibility.
- “For frameworks” tier not enforced by compiler - this tier would need to be enforced in the same way as in the proposed approach, increasing the amount of work.
- Intra-module dependencies - due to some modules depending on other modules, it may necessitate multiple refactors, further compounding the number of changes needed.

Another approach would be to use opaque pointers and the pointer-to-implementation pattern. This is unfeasible for this project, due to the scale of the refactoring needed and effectively breaking each entry point. This pattern additionally introduces performance overhead, due to additional allocations and pointer indirection.

## Adoption strategy

To ensure a smooth transition, the rollout of the enforcement of the reduced API surface will follow a phased approach, beginning with a grace period.

Phase 1:

The first React Native version introducing the new umbrella headers will have the enforcement mechanisms disabled by default. Both the umbrella headers and individual ones will remain accessible simultaneously for all consumers. Library maintainers and application developers will be able to test their codebase by opting in for the strict checks by enabling `RN_STRICT_API` build flag. The exact duration of this grace period is yet to be determined.

Phase 2:

After the grace period ends, following React Native releases will enforce the new three-tier visibility model. Because the enforcement is configured by a build flag, it will be technically possible to disable it as an escape hatch, however no compatibility or stability guarantees will be given for non-public symbols from that point onward.

## How we teach this

Once the project is rolled out, we will prepare a blog post to be published along the React Native version announcement, similarly to [how it was done for the JS Stable API project](https://reactnative.dev/blog/2025/06/12/moving-towards-a-stable-javascript-api). In the blogpost we will explain the rationale behind the changes, what exactly changed, how to migrate to the new model, and how to enable the enforcement mechanism.

Similarly [to the JS Stable API project](https://github.com/react-native-community/discussions-and-proposals/discussions/893), we will also prepare a place to collect the community feedback about the new entry points during the grace period.
