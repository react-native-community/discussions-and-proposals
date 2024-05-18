---
title: React Native Framework
author:
- Nicola Corti
date: 2024-02-02
---

# RFC0759: React Native Frameworks

> [!IMPORTANT]
> This RFC is open for comments till Monday 19th February 2024.

## Summary

This RFC defines the concept of **React Native Frameworks**: frameworks built on top of React Native to provide production-ready applications. With this document, we intend to declare what are the responsibilities of React Native itself vs. responsibilities of React Native frameworks.

By defining what React Native is and is not responsible for, we can improve the focus and stability of React Native and ensure that reliable extension points are available where needed.

We’re formalizing this concept now as, ultimately, we want developers to build **production-grade applications** while experiencing a **top-notch developer experience**.

We believe most of our developers **should be using a React Native Framework**, and this document explains what exactly this means and how we can achieve this goal.

## Table of Content

- [RCF0759: React Native Frameworks](#rcf0759-react-native-frameworks)
  - [Summary](#summary)
  - [Table of Content](#table-of-content)
  - [What is a React Native Framework?](#what-is-a-react-native-framework)
  - [Why are we defining this now?](#why-are-we-defining-this-now)
  - [Frameworks \& Platforms](#frameworks--platforms)
  - [Responsibilities](#responsibilities)
    - [What does the React Native package do](#what-does-the-react-native-package-do)
    - [What does the React Native package not do](#what-does-the-react-native-package-not-do)
  - [What do we recommend to React Native app developers?](#what-do-we-recommend-to-react-native-app-developers)
  - [What do we recommend to React Native library developers?](#what-do-we-recommend-to-react-native-library-developers)
  - [Becoming a React Native Framework](#becoming-a-react-native-framework)
    - [Benefits](#benefits)
    - [Requirements](#requirements)
      - [DOs of Frameworks](#dos-of-frameworks)
      - [DON’Ts of Frameworks](#donts-of-frameworks)
      - [Note for Platforms developers](#note-for-platforms-developers)
  - [Evolving the React Native Community Template \& CLI](#evolving-the-react-native-community-template--cli)
    - [Template](#template)
      - [Internal Use Cases](#internal-use-cases)
      - [External Use cases](#external-use-cases)
        - [Developers that intend to migrate to an existing React Native Framework](#developers-that-intend-to-migrate-to-an-existing-react-native-framework)
        - [Developers that can’t migrate to an existing React Native Framework because it doesn’t satisfy their requirements](#developers-that-cant-migrate-to-an-existing-react-native-framework-because-it-doesnt-satisfy-their-requirements)
      - [Community CLI](#community-cli)
        - [Commands to be moved inside react-native core](#commands-to-be-moved-inside-react-native-core)
        - [Commands to be removed](#commands-to-be-removed)
        - [Commands to be left inside @react-native-community/cli](#commands-to-be-left-inside-react-native-communitycli)
        - [Commands to be extracted as standalone commands](#commands-to-be-extracted-as-standalone-commands)
    - [The init command](#the-init-command)

## What is a React Native Framework?

React Native allows developers to use the [React programming paradigm](https://react.dev/learn/thinking-in-react) to ship applications to native targets. The React Native team is focused on creating **core APIs** and **functionalities** that fit the most general use case when developing native apps (not just restricted to Android & iOS).

Shipping native apps to production usually requires a set of tools & libraries that are not provided by default as part of React Native, but are still crucial to hit the ship to production. Examples of such tools are: support for publishing applications to a dedicated store, or support for routing and navigation mechanisms.  

When those tools & libraries are collected to form a cohesive framework built on top of React Native, we call this a **React Native Framework**.

An example of Open Source React Native Framework is [Expo](https://expo.dev/).

If you’re building production-ready applications without using React Native Framework, you probably have a collection of scripts and automations that helps you develop and ship applications to production. This collection of infrastructure is a bespoke React Native Framework that you built around your own needs and processes. For this reason, is easier to think about React Native Framework as:

> **You’re either using a React Native Framework or you’re building your own React Native Framework**

A practical example of bespoke React Native Framework is how Meta uses React Native internally. We do have a collection of libraries, scripts and tools that make the integration of React Native easier with the various apps where it's used.

For the rest of this document we will clearly state what are the responsibilities of React Native Frameworks, so that if you don’t want to use an available solution, you know what are the boundaries of React Native.

> [!NOTE]
> We do acknowledge that “Framework” is an overloaded term here, as React Native itself is a UI framework which uses React. For the sake of simplicity in the rest of this document, the “Framework” term will always refer to a React Native Framework — a set of tools & libraries built on top of React Native.

## Why are we defining this now?

You might be wondering why we are defining those concepts now. React Native has been **growing in popularity, scope & maturity** over the recent years. This is mostly thanks to one of the strongest components of React Native: our **ecosystem**.

The Authors & Maintainers of React Native libraries and tools have played such a crucial role in the development of React Native.

The intention of this document is **not** to drop responsibilities on their shoulders. The React Native team is fully committed to supporting React Native in the same capacity as we’ve been doing to this day. We believe that by clearly defining our **boundaries**, we could build a **better ecosystem** for everyone and foster further growth to increase adoption and stability of React Native.

This is one of the **goals** we set ourselves when defining what a React Native Framework is, among others:

- **Allow developers to “think in React”** - Each library, Framework or tool should sit on top of the core fundamental APIs, so that it can be easily reused and ported between different targets.
- **Providing great APIs & extension points** - It’s crucial for Framework developers to have a way to voice their concern and share how they’re integrating with React Native and what does/doesn’t work for them. Frameworks should be able to “plug” into React Native without having to fork or use private APIs. Relying on public and stable APIs will provide a more sustainable future and an easier update experience for framework authors.
- **Set “the standard” for the ecosystem** - React Native should set standards that are general enough to work across Frameworks so that Frameworks can use our standards as a foundation to build tools and extensions on top of them.
- **Enable portability** - React Native should provide all of the standards and language features needed to write libraries that are portable to use across all frameworks (either bespoke or Open Source Frameworks).
- **Fostering a healthy community** - As mentioned, by defining our boundaries, it will be clearer where we draw the line between React Native Core APIs and third-party framework APIs, we’ll save a lot of confusion for developers, and we’ll avoid re-implementing solutions.

## Frameworks & Platforms

Before we delve into the responsibilities of React Native & Frameworks, it’s crucial to define what is a **Platform**, as one of the key value propositions is to let you write applications and target several different Platforms.

A platform is the underlying computing architecture and operating system on which applications run.

React Native offers **first-party support** for Android & iOS, as the code to target Android & iOS lives inside the [react-native repository](https://github.com/facebook/react-native). Such code is developed and maintained by Meta.

Over the years, the ecosystem has developed support for other platforms which we call [Out-of-Tree platforms (OOT)](https://reactnative.dev/docs/out-of-tree-platforms).

Each OOT platform comes with its own documentation, setup instructions, and tools/libraries to develop applications on the new target platform.

We consider the Framework and Platform concepts as **orthogonal**, i.e. you can use a Framework to ship to multiple platforms, and a Platform can be targeted by multiple Frameworks. In this context the role of a Framework is to abstract complexity and make it easier to ship to multiple Platforms providing a cohesive Developer Experience (see below).

## Responsibilities

This section describes where we draw the line between the React Native package and Frameworks.

> [!NOTE]
> For the sake of simplicity, we will be focusing on the React Native package (i.e. the npm package called `react-native` which offers first-party support for Android & iOS).

### What does the React Native package do

The React Native package is responsible for **core APIs and capabilities** to build native apps. React Native is also responsible for offering **APIs** that allow Frameworks to deliver their experience. When such APIs are missing, it’s our interest to create them and have a conversation with Framework developers to inform their development, and help framework developers merge changes back into React Native.

When a new significant change to the API is needed, especially if it impacts Frameworks & libraries, we'll be publishing a dedicated RFC to discuss the change.

React Native offers first-party support for Android & iOS. Other platform support is offered by other libraries or Frameworks.

Here is a (non-exhaustive) list of areas of responsibilities of the React Native package:

- **Rendering & Components** - The React Native package is responsible for rendering on screen and offering basic building blocks to build custom UI. Parts of this API surface are:
  - The Native Renderer API & Native Component API
  - APIs of Core Components provided by React Native
  - Accessibility support for such components
- **Native Modules** - The React Native package is responsible for offering a mechanism and an API to access the underlying target platform. Parts of this API surface are:
  - The Native Modules API
  - APIs of Core Native Modules provided by React Native
- **Runtime & Initialization** - The React Native package is responsible for offering an API to get access to the runtime and defining its lifecycle. The bridge and the bridgeless API are part of this API surface.
- **Internals mechanism** - The React Native package is responsible of encapsulating away complexity and handling mechanisms such as:
  - The threading model
  - Interoperability layers between different architectures
- **JS Engine Integration** - The React Native package is responsible for offering an API to integrate a target JS engine. React Native is distributed with built-in integrations for Hermes and JavaScriptCore. Third-party engines (like V8) may be integrated but are not directly supported.
- **Bundler integration** - The React Native package defines the integration with Metro, the React Native bundler. Part of this API surface are:
  - Building the JS and asset bundle
  - The development server
  - Debugging integrations
  - Sourcemap support
  - Error and Crash reporting
- **Native build integration** - The React Native package is responsible for defining how the Android & iOS final artifact are built, as well as how Java/Kotlin/Swift/ObjC/ObjC++ and C++ should be compiled and distributed.
- **3rd party libraries** - The React Native package is responsible for defining integration API for 3rd party libraries (i.e. how to create libraries for React Native and how to integrate them with the runtime). Frameworks can build on top of those APIs to simplify the library integrations and discovery via mechanisms such as auto-linking.

### What does the React Native package not do

React Native Frameworks provide features that **most native apps** that are intending to reach production eventually need.

Here is a (non-exhaustive) list of areas of what the React Native package is **not directly focusing** on, and is delegating this responsibility to frameworks:

- **Project Scaffolding** - Frameworks can offer tools and templates to create new projects so that developers can get started developing in a matter of seconds.
- **Tooling for React Native upgrade DevX** - Similarly to project scaffolding, framework can offer tools to make sure the project can easily keep up with the latest React Native version. Tools such as the upgrade helper, Expo Config Plugins and others fit in this category.
- **Navigation and Routing solutions** - React Native doesn’t offer a navigation or a routing solution out of the box. Frameworks can either offer their own solution or recommend a preferred navigation solution to use.
- **Managing the build experience** - Framework can offer tools to simplify the native builds and move to fully managed native builds that are happening in the cloud.
- **Manage the publishing experience** - Frameworks can offer solutions to publish applications to public stores (App Store/Play Store and others) either locally or managed in the cloud.
- **Providing OTA updates** - Frameworks can offer mechanisms to deploy Over-the-air updates to app users. Tools such as EAS Update & Codepush fall into this category.
- **Auto-Linking libraries** - Frameworks can offer mechanisms to simplify linking of external 3rd party libraries.
- **Support new targets via Out-of-tree Platforms** - Frameworks can offer support for additional targets by using OOT Platforms, in order to offer a more cohesive development experience.

## What do we recommend to React Native app developers?

As mentioned before, we believe a React Native Framework offers all the tools you’ll eventually need to get to production quickly while offering a superior developer experience. As we mentioned before:

> You’re either using a React Native Framework or you’re building your own React Native Framework

We believe that an existing React Native Framework should be the default solution for the **majority of app developers**, including developers who have more of a web background than a native background.

You should be building your own React Native Framework **only** if you’re:

- An expert native engineer with requirements not satisfied by existing frameworks
- A React Native power-user which needs to integrate React Native inside their organization with custom workflows.
- A developer looking into porting React Native to newer Native platforms (e.g. visionOS).
- A company that for specific requirements (e.g. legal, licensing) can't use one of the existing Frameworks.

For this reason we intend to **recommend** developers to use an existing React Native Framework. At the time of writing, Expo is the only Framework in the ecosystem that fits the definition we shared above, and so we will recommend using Expo for most developers.

We’re going to update our official channels (websites, GitHub, social media channels) with the recommendations explained in the following paragraphs.

## What do we recommend to React Native library developers?

React Native Frameworks can build **Framework specific APIs** on top of the Core APIs provided by the React Native package (i.e. the Components and Modules API), to deliver a better developer experience for their users. A notable example of this is the Expo Modules API.

App developers can greatly benefit from such abstractions which can **reduce** the steep **learning curve** of some of the core React Native APIs.

However, library developers could have different requirements in terms of which programming languages they intend to use and which platform they intend to support.

The **current recommendation** to library developers is as follows:

- If you intend to support Out-of-tree platforms:
  - Refer to the documentation of the specific Out-of-tree platform on how to create libraries for that platform
- If you intend to integrate with C++ (i.e. you have a native library to integrate with or you want to share implementation among platforms):
  - Use the React Native Core API, as they provide easier access to lower-level mechanisms.
- If you’re looking for a better developer experience & are willing to require that your consumer depend on a Framework specific dependency:
  - Use the Framework specific API.
  - The tradeoff here is on simplicity/ergonomics vs wider compatibility.
    - Frameworks are able to quickly ship and iterate on Developer Experience improvements to the Core APIs (e.g. build an idiomatic API for modern languages such as Swift/Kotlin). The best ideas can later be pulled into the React Native Core APIs once they are proven, if they are a good fit.
    - On the other hand, using Framework specific APIs requires library consumers to add an extra dependency in their application on the specific Framework. This could result in libraries that are less compatible with apps that are using other Frameworks.
- Otherwise if you don’t want to impose a Framework dependency on their consumer, to ensure that your library is compatible with all Frameworks:
  - The recommendation is to use the React Native Core API.

We do acknowledge that the current recommendation for library authors is complicated, and the React Native team will be working with Frameworks developers in the future to simplify the current offerings.

## Becoming a React Native Framework

In this section, we clarify what are the **requirements** to become an official React Native Framework. We call this also the **Framework Contract** as it describes what are the benefits of- and the asks for being an official Framework.

### Benefits

Frameworks that are adhering to requirements described below:

- Will be spotlighted in the Getting Started page of the React Native website (our highest traffic page). Specifically, we’ll recommend to developers to use one of the Frameworks which satisfies the requirements.
- Will be mentioned during public comms as one of the recommended frameworks by the React Native core team.
- The company/entity behind the Framework will automatically be granted a [partner status](https://github.com/facebook/react-native/blob/main/ECOSYSTEM.md#partners).
- PRs and Issues coming from the Framework developers will receive higher priorities.

### Requirements

Here is a short summary of what are the requirements for React Native Frameworks.

#### DOs of Frameworks

- **Focus on developer & user experience** - We believe Frameworks should aim to provide a top-notch developer experience. Frameworks that fail to provide good documentation or developer workflows are not likely high quality frameworks, and developers will most likely not want to use them. Similarly, we want to promote frameworks that provide a top-notch app user experience and do not ignore basic user needs, such as accessibility, performance, and delightful UIs.
- **Focus on building cohesive experiences** - Developers are looking into deploying universal apps across multiple targets. We’re delegating this responsibility to frameworks that can provide better experiences to app users and craft cohesive developer experiences between mobile, web, desktop and beyond.
- **Focus on abstracting complexity away** - Any initiative that simplifies the developer workflow is welcomed in the framework domain.
- **Be popular** - We’ll only be recommending solutions that are widely known and adopted by the community.
- **Open source** - Recommended frameworks must be free, open source, projects. We allow for companies to provide paid services on top of the Open Source framework, but these services should not be a requirement to use the framework and should not have vendor lock-in.

#### DON’Ts of Frameworks

- **Do not hard fork react-native** - As a rule of thumb, you should not GitHub-fork react-native and re-publish it. If you’re doing so, it’s probably due to a lack of APIs that React Native should provide. Please reach out to us and attempt to de-fork your framework.
- **Do not soft fork the core APIs** - We want to make sure developers need to learn only once how to write a native component. Building abstraction and wrappers is accepted, forking is not, as it makes it harder for developers to share components/modules between targets.
- **Do not fork the ecosystem** - It’s crucial that `react-native-*` libraries can be easily shared and ported to other targets or frameworks (with little to no work). Frameworks should aim at building libraries that work across the whole ecosystem and not solutions that silo the developers away.
- **Do not redistribute prebuilts** - React Native ships native prebuilts for several targets. Do not attempt to republish them as this reduces security for app users and is essentially yet another fork.
- **Do not depend on the internals** - Do not depend on internal APIs. As a rule of thumb, depending on private/internal API (i.e. those from the `internal` package or with `unstable_` prefix) is not recommended and will cause breakages in the future. Specifically, we might consider rotating private APIs (changing the name) in the future to discourage its usages. When there is a need for Frameworks to access those internal APIs, we’ll be annotating them to make them easy to recognize it (i.e. with the `@FrameworkAPI` annotation).

#### Note for Platforms developers

The aforementioned Do/Dont’s don’t apply strictly to **Platforms developers**. While they do represent general good practices in the software development field, targeting a new platform poses a series of **unique challenges**.

Specifically, when supporting platforms, having a fork might actually be the preferred solution, based on the amount of custom logic that is necessary to maintain. Similarly, certain platforms require redistributing prebuilts for new binary architectures which are not already provided by React Native.

The React Native team is committed to fostering innovation and supporting new platforms by offering APIs and artifacts for those new platforms, whenever possible.

## Evolving the React Native Community Template & CLI

Developers who have been using the React Native **Community CLI & Template**, might be wondering how React Native Frameworks affect them.

We want to reassure React Native developers that are using those tools and clarify that those tools will **still be supported** **and updated**, and you can keep on using them if you wish. However, we’ll be recommending new developers and **newly created mobile apps** to use a React Native Framework from now on. Similarly, you’re free to migrate to a React Native Framework if you intend to.

We do recognize that both the Community CLI & Template are crucial pieces of infrastructure of the current React Native experience. At the same time, they’re highly controversial as their role is not clearly defined between the React Native package and Frameworks.

Specifically, we feel that those tools are attempting to address **two different user groups** (internal developers and external developers) with different requirements, unable to solve either as well as a focused tool could. Those two groups would be much better suited by fundamentally different approaches, while the current setup ends up delivering a suboptimal experience for both.

We believe that a significant number are approaching React Native via the Community CLI as they believe this is the official recommendation (as it is commonly referred to as the React Native CLI for short), and they would be better suited by using a React Native Framework.

Ultimately, developers experiencing the template/CLI as they exist today is one of the causes of the **largest negative impact** on React Native's perception, adoption, and future growth. This is reflected by the adoption drop of the React Native Community CLI in the latest [The State of React Native 2023 Survey](https://results.stateofreactnative.com/tools/tools_section_streams). Moreover in the State of React Native 2022, the majority of developers [reported Upgrades as #1](https://results.2022.stateofreactnative.com/opinions/) pain point in the ecosystem, as the Template/CLI in the current state are not resolving this crucial gap in the developer experience.

### Template

The React Native template is located inside the `/template` folder of the NPM package ([visible here](https://github.com/facebook/react-native/tree/main/packages/react-native/template)) and offers a mechanism to create a simple Android & iOS app which runs React Native.

A new project can be scaffolded from the template by using the community CLI, and the `npx react-native init` command (which delegates to` @react-native-community/cli init `behind the scenes).

Practically, the template represents a **minimal integration** of the React Native **APIs**. Those APIs are partially described in the “[Integrating with Existing Apps](https://reactnative.dev/docs/integration-with-existing-apps)”. In an ideal world, if you were to:

1. Create a new Android/iOS project on Android Studio/Xcode
2. Run through [all the steps in this page](https://reactnative.dev/docs/integration-with-existing-apps).

you will effectively **recreate the template**.

The template gets more controversial as it’s **not a Framework** but it offers some of the capabilities that are specific to Frameworks. For example it offers an auto-linking implementation, and a mechanism to help with upgrades (the upgrade helper). It lacks other crucial features of frameworks, such as routing, navigation or support for publishing apps.

To fully understand the role of the template, it’s crucial we understand what are the use cases of the template as of today.

The React Native template is used for:

1. **Internal** use cases (Meta engineers/Contributors)
    1. The React Native CI is using it to test that a new project can be successfully created on every PR/Commit ([example job definition here](https://github.com/facebook/react-native/blob/28dfdb22ebaff7055fd8bb4b8847e018181f849d/.circleci/config.yml#L769))
    2. The Release Crew is using it to test that a new app can be created, built and executed correctly **before releasing** a new version of React Native ([docs on this process are here](https://reactnative.dev/contributing/release-testing))
    3. Issues and bug reports require a **reproducer**, being either an Expo Snack or an instance of a template project ([see here for the reproducer](https://github.com/react-native-community/reproducer-react-native)).
2. **External** use cases
    1. React Native developers that are **not using a Framework** because they haven’t migrated to it yet
    2. React Native developers that are **not using a Framework** such as Expo because it doesn’t satisfy their requirements
    3. New React Native developers that are creating a new app. They might either not know how to use a Framework, or they might seek the official/bare/**Meta-recommended** way to use React Native.
    4. Developers looking into integrating React Native into frameworks or existing apps. They can use the template as a **reference** for their own integration, or to understand how the internals work.

As we do have a mixture of internal and external use cases for the template, the proposed solution is to **split the template** between internal and external use cases.

#### Internal Use Cases

For internal use cases, we’ll be creating a **HelloWorld App** inside the React Native repo (fun fact: the template is actually called `HelloWorld` till it gets scaffolded by the community CLI).

The HelloWorld App is effectively a new React Native app created from the current template, and **committed** inside the repo.

This HelloWorld App will be similar to RN-Tester but with a couple of crucial differences:

- It will **not build from source**, it will be configured to use a packaged version of React Native (via local repositories such as Verdaccio, Maven Local, etc.)
- It will **not contain examples**, but just a landing page, making it easier to start and work with.
- Can be used as a **reproducer to verify bugs against main**. i.e. developers can easily git clone the `react-native` repo and use the HelloWorld App to reproduce bugs.
  - This is similar to the internal **Playground app** that Meta has to offer reproducers.
- It can be used by the release crew to **test blank apps** with newer React Native releases.

For reproducers using numbered versions of React Native, we can either:

- Ask developers to git clone react-native, checkout the release branch, and update the HelloWorld app (potentially impractical and hard for everyday users).
- Adapt the [reproducer-react-native](https://github.com/react-native-community/reproducer-react-native) GitHub repo to use the HelloWorld App from a given branch/version.

Moreover, the HelloWorld app can be used by framework authors and advanced developers as a reference for React Native integrations (point _2.d_ of External Use Cases)

- It intentionally **won’t** be possible to scaffold new apps from it. Developers will have to manually copy the files, making it impractical for everyday use.

#### External Use cases

As mentioned above, we want to keep on supporting existing developers that are currently using the React Native template.

We won’t be removing the template, as developers are relying on it via tools such as the [upgrade helper](https://react-native-community.github.io/upgrade-helper/), which are part of their normal workflow. Removing/deprecating those tools without a proper replacement is **unacceptable**.

Those developers can either migrate to an existing React Native Framework or keep on using the template, as described below:

##### Developers that intend to migrate to an existing React Native Framework

Those developers have no practical blockers to use a React Native Framework, they simply haven’t had the time or don’t feel that those Frameworks are ready for their use case yet.

For those developers we will:

- Update the guidance on the website mentioning to use a React Native Framework
- Update the community CLI and other developers tools to mention the new recommendation around React Native Frameworks.
- Work with existing React Native Framework to identify use cases that are not properly covered by their Framework yet
- Work with existing React Native Framework to provide a viable migration path from the template.

##### Developers that can’t migrate to an existing React Native Framework because it doesn’t satisfy their requirements

We do have several developers who can’t use a Framework yet, due to specific constraints.

One of the partners that we had conversation with, is developing React Native apps inside a monorepo with multiple native apps (a mixture of RN-only and mixed native apps), as well as multiple React web applications. They built their own bespoke Framework, but still rely on the template & upgrade helper as a reference.

Another scenario is developers that are building apps targeting a number of different paltforms. At the time of writing, not all the platforms are supported by existing Frameworks (e.g. Expo doesn't support shipping applications to Windows). Those developers could resort to either building their own bespoke Framework, or using a mixture of existing Frameworks and custom infrastructure to support the necessary platforms.

For those developers we will:

- Move the current template **outside** of the `react-native` package:
  - Specifically move it to `@react-native-community/template` and kept on being **maintained by the community and the React Native team**.
    - This is the least disruptive change for developers, as it will keep on being easily accessible but it will be clearer that is a community owned component, and not part of the core of react-native
    - Moreover, the template will be closer to where the community CLI lives today (`@react-native-community/cli`)
  - We (Meta & the community) want to keep on **maintaining** the template and make sure that developers that can’t migrate to a framework, still have a viable alternative.
  - We don’t foresee the template to change drastically in the future. Only **minimal changes** will be allowed to the template, to reduce the upgrade pain between versions.
  - Similarly, we’ll aim at **reducing the breaking changes** as much as possible.
- Update all the various community tools to refer to the new template location (e.g. the Upgrade Helper)
- Work with existing React Native Framework to identify use cases that are not properly covered by their framework

#### Community CLI

The React Native Community CLI is tightly coupled to the template. Apps like RN-Tester (and the proposed HelloWorld app) are using the community CLI to build and being tested on debug/release on CI. Similarly the community CLI is used to do release testing.

On top of this, having the community CLI living in a separate repository (`@react-native-community/cli`) introduces a number of **circular dependencies** and friction as part of the release process.

Similar to the Template, we envision having a **private CLI** inside the react-native repository to support the internal use cases mentioned above. This will be achieved by evolving each command of the CLI as follows:

##### Commands to be moved inside react-native core

Commands that live inside `react-native` should be extensible enough to be **used by Frameworks or libraries, **if they wish to**.**

- `clean, build-android, build-ios, run-android, run-ios, log-ios, log-android`
  - Those are wrappers around the native build system and need to be moved inside the react-native repository.
- `start, bundle`
  - Those are wrappers around Metro and [are effectively moved](https://github.com/facebook/react-native/pull/38795) inside the `react-native` repo already.
- `profile-hermes`
  - Should be moved inside react-native as part of default debugging/profiling experience with Hermes

##### Commands to be removed

- `upgrade`
  - This command has been removed and now only points to the upgrade helper
  - Potentially can be removed entirely
- `ram-bundle`
  - Can be removed as not relevant anymore

##### Commands to be left inside @react-native-community/cli

- `config` and Auto Linking of dependencies.
  - Those should **not** be moved inside react-native.
  - They should live alongside the template as the template is relying on the community CLI auto linking logic

##### Commands to be extracted as standalone commands

- `info` and `doctor`
  - Those commands should be moved out and become standalone utilities and should be **working well across frameworks**
  - Those are key parts of the React Native **bug and issue reporting** experience so should be easily accessible for developers.
  - As an alternative they could be moved inside `react-native` and shipped as separate packages that developers can selectively use if they wish.

### The init command

The `init` command needs a dedicated section as it’s the entry-point of most of our developers to React Native.

We suggest to evolve the init command as follows:

- **react-native@0.74**: Inform the developers of the underlying command they’re invoking:

```bash
$ npx react-native@latest init 

Running: npx @react-native-community/cli init

... logs as usual
```

- **react-native@0.75**: Inform the developers that init is deprecated, and recommend replacements (either use a Framework, with link to documentation or fallback to the Community CLI init command). The command will proceed to run the Community CLI init command in order to preserve existing behavior:

```bash
$ npx react-native@latest init 

⚠️ The `init` command is deprecated. 
The behavior will be changed on January 1, 2025 (344 days). 

- Switch to npx @react-native-community/cli init for the identical behavior.
- Refer to the documentation for information about alternative tools: https://reactnative.dev/docs/getting-started

Running: npx @react-native-community/cli init 

... logs as usual
```

- **react-native@>=0.76**: Inform the developer that the command has been deprecated and just fire a warning for alternatives

```bash
$ npx react-native@latest init 

⚠️ The `init` command is deprecated. 
The behavior will be changed on January 1, 2025 (344 days). 

- Switch to npx @react-native-community/cli init for the identical behavior.
- Refer to the documentation for information about alternative tools: https://reactnative.dev/docs/getting-started
```

Similarly, the `create-react-native-app` command (currently owned by Expo) will be updated to just print a warning to developers by React Native **0.75**:

```bash
$ npx create-react-native-app@latest

Need to install the following packages:
create-react-native-app@2.0.0
Ok to proceed? (y)

⚠️ This tool does not initialize new React Native projects.

It's recommended to use a framework to build apps with React Native, for example:

Expo
     npx create-expo-app@latest

React Native Community template
     npx @react-native-community/cli@latest init

Learn more: https://reactnative.dev/docs/getting-started
```

A separate `init-internal` (temporary name) will be developed to support **internal use cases** of React Native, mentioned in the [Template section of this RFC](#template).
