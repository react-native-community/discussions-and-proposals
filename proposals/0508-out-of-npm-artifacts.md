---
title: Out-of-NPM package solution for React Native Artifacts
author:
- Nicola Corti
date: 2022-09-09
---

# RFC: out-of-NPM package solution for React Native Artifacts

## Summary

The `react-native` NPM package unpacked size is now 126 Mb. We’re close to hit the NPM package size limit.
We need to find a solution to store native artifacts in the long run.

## Motivation

With React Native 0.69, the `react-native` NPM package unpacked size is now 126 Mb. 
React Native 0.68 had an unpacked size of 35.6 MB. 

We’re close to hit the NPM package limit and we were already forced to remove Hermes debug symbols for Android to ship the package to NPM.

Publishing a ~250Mb NPM package is failing with:

```
`npm notice 
npm ERR! code E413
npm ERR! 413 Payload Too Large - PUT `[`https://registry.npmjs.org/tfjs-node-lambda-releases`](https://registry.npmjs.org/tfjs-node-lambda-releases)` - Payload Too Large
Error: Process completed with exit code 1.`
```

Essentially NPM is trying to Base64 the package in memory and publishing it to the NPM registry with an HTTP `PUT`. 
This is failing as the webserver received a too long payload.  More on this here [https://github.com/npm/npm/issues/12750](https://github.com/npm/npm/issues/12750)

We’re particularly hit by this problem as we're shipping native built libraries inside a NPM package.

The biggest offender in terms of size are:

* the `/android` folder which contains prebuilt for both `react-native` and `hermes-engine` used by Android Users on the Old Architecture (which are not building RN or Hermes from source) - ~66Mb
    * The `/android` folder is essentially a Maven Local repository that is following the typical Maven Folder structure `com/facebook/react/react-native` follows the `<group-id-package>/<artifact-id>`
* the `/sdks ` folder which contains prebuilt `hermesc` for the Hermes Compiler - ~43Mb

![react-native npm package size graph](/assets/npmsizegraph.png)

Having all the artifacts shipped inside a single NPM package, has also the side effect of forcing all users to re-downloading the same ~100Mb archive even if they’re not using some of the features (e.g. you’ll be downloading the Android archives even if you’re just building an iOS app).

## Overview of current artifacts

In order to have an holistic overview of which artifacts we have at the moment and where they live, let’s walk through each of them:

### The React Native Android Archive

Currently we ship a prebuilt version of React Android inside the `react-native` NPM package. The prebuilt is located inside the `android` folder and needs to follow this structure:

```
// Checksum files have been removed for brevity.
android
└── com
    └── facebook
        └── react
            └── react-native
                ├── 0.70.0
                │   ├── react-native-0.70.0-debug-sources.jar
                │   ├── react-native-0.70.0-debug.aar
                │   ├── react-native-0.70.0-release-sources.jar
                │   ├── react-native-0.70.0-release.aar
                │   ├── react-native-0.70.0.module
                │   └── react-native-0.70.0.pom
                ├── maven-metadata.xml
                ├── maven-metadata.xml.md5
                ├── maven-metadata.xml.sha1
                ├── maven-metadata.xml.sha256
                └── maven-metadata.xml.sha512
```

The `android` folder is effectively a Maven Repository which follows the [official repository layout](https://cwiki.apache.org/confluence/display/MAVEN/Remote+repository+layout). The `.module` file is a [Gradle Module Metadata](https://docs.gradle.org/current/userguide/publishing_gradle_module_metadata.html) file. It allows to publish two variants for debug and release.
The .aar files contain the Android compiled bytecode for React Native, alongside native libraries (.so).

At this stage, debug symbols for React Native are not distributed (also for the `react-native-0.70.0-debug.aar`) due to constraints in size for the NPM package. Similarly, prefab support is not enabled (i.e. headers are not distributed, libraries can’t depend on native libraries directly and are forced to unzip the .aar manully - See [https://github.com/react-native-community/discussions-and-proposals/issues/423](https://github.com/react-native-community/discussions-and-proposals/issues/423)).

The `-sources.jar` file contains Java/Kotlin sources used by IDEs to offer code visualization support.
Alongside those files, each file is distribute with its own checksums (.md5, .sha1, .sha256, .sha512).
GPG Signatures for those files are not distributed (i.e. `.asc` files).

Please note that the files needs to follow the repository layout to properly be consumed.

### The Hermes Android Archive

With [Bundled Hermes](https://reactnative.dev/architecture/bundled-hermes), the prebuilt version of Hermes have been moved inside the `react-native` NPM package. They’re located inside the `android` folder similarly to:[The React Native Android Archive](#the-react-native-android-archive)

```
// Checksum files have been removed for brevity.
android
└── com
    └── facebook
        └── react
            └── hermes-engine
                ├── 0.70.0
                │   ├── hermes-engine-0.70.0-debug-sources.jar
                │   ├── hermes-engine-0.70.0-debug.aar
                │   ├── hermes-engine-0.70.0-release-sources.jar
                │   ├── hermes-engine-0.70.0-release.aar
                │   ├── hermes-engine-0.70.0.module
                │   └── hermes-engine-0.70.0.pom
                ├── maven-metadata.xml
                ├── maven-metadata.xml.md5
                ├── maven-metadata.xml.sha1
                ├── maven-metadata.xml.sha256
                └── maven-metadata.xml.sha512
```


Due to limitation on the NPM package size, we’ve been forced to **remove the debug symbols & disable prefab for Hermes** (see [https://github.com/facebook/react-native/pull/33439](https://github.com/facebook/react-native/pull/33439)) from the `react-native` NPM package. 

This had a couple of negative side effects:

* Android Users **on New Architecture** could **not** use a pre-built of Hermes (as we can’t distribute the prefab). So they’re forced to **build Hermes from source**.
    * The impact is that this is creating a discrepancy between the Old and the New Architecture resulting in generally a degraded Developer Experience for the New Architecture.
* Android Users on Old Architecture are forced to download the debug symbols from CircleCI, as they’re beyond 200Mb and can’t be distributed inside the NPM package.
    * The impact is that users experiencing native crashes on Hermes don’t see symbolicated stacktraces (on both Old/New Arch).

### The Hermesc Binary

With [Bundled Hermes](https://reactnative.dev/architecture/bundled-hermes), The Hermes Compiler Binary `hermesc` is located inside the `sdks/hermesc` folder of `react-native`

```
sdks
├── ...
└── hermesc
    ├── linux64-bin
    │   └── hermesc
    ├── osx-bin
    │   └── hermesc
    └── win64-bin
        ├── hermesc.exe
        ├── icudt64.dll
        ├── icuin64.dll
        ├── icuio64.dll
        ├── icutest64.dll
        ├── icutu64.dll
        ├── icuuc64.dll
        ├── msvcp140.dll
        ├── vcruntime140.dll
        └── vcruntime140_1.dll
```

`hermesc` is a binary that should be executed by the host machine at build time when preparing a release version of an app (Android/iOS) that uses Hermes.

### The Hermes iOS Tarball

With [Bundled Hermes](https://reactnative.dev/architecture/bundled-hermes), we publish a prebuilt version of the Hermes Runtime for iOS. This allows users to use Hermes on iOS without having to build it at all (both on Old and New Architecture).

As the prebuilt tarball is 466 MB, it goes beyond the NPM package limit.

To overcome this limit, the prebuilt Hermes Tarball is uploaded on Github Releases (example: https://github.com/facebook/react-native/releases/tag/v0.69.4)

The URL to download the tarball is deterministic and uses the [version key to reference it](https://github.com/facebook/react-native/blob/0b4b7774e2d71259962ed36b7acb5c3989c3be9c/sdks/hermes-engine/hermes-engine.podspec#L39):

```ruby
 source[:http] = "https://github.com/facebook/react-native/releases/download/v#{version}/hermes-runtime-darwin-v#{version}.tar.gz"
```

### The React Native Gradle Plugin

The [React Native Gradle Plugin](https://www.npmjs.com/package/react-native-gradle-plugin) is distributed from sources as a NPM package ([source of the plugin are here](https://github.com/facebook/react-native/tree/main/packages/react-native-gradle-plugin)).
Users are consuming the plugin as an Included build: [codepointer](https://github.com/facebook/react-native/blob/4eec47344e25b2e2cb2e6dbdb3f989ed5b6e2747/template/android/settings.gradle#L4).

This is impacting the DevX mostly because:

* Users are forced to recompile the source code of the plugin at every first build (slower build time).
* Users are exposed to build warnings for the plugin, without having a way to hide them.
* Any change to the plugin (i.e. a user touching any of its source file) will cause a whole rebuild of the Android project, due to invalidation of the build classpath.

Ideally the Gradle Plugin should be bundled following the Maven Repository layout similarly to all the others packages (react-native and hermes-engine).

### Other Android/iOS Prebuilt Dependencies

In the future, we might want to add more library/artifacts, to potentially modularize the project. 
An example is the Module Annotation Processor ([see this PR](https://github.com/facebook/react-native/pull/25922)).

As of today, we don’t have a way to distribute this annotation processor. It should either be distributed from source (similarly to the React Native Gradle Plugin) or as prebuilt (similarly to React Native).

## Scenarios to consider

When solving this issues we should consider several scenarios. Each scenario is distinguished by a different `version` in the top level `package.json`

* **Building from main**
    * Version in the top level package.json: `1000.0.0`
    * This is the scenario you’re faced when you `git clone` React Native.
    * In this scenario, everything should be **built from source**. There should be no available artifacts.
* **Building a numbered version of React Native**
    * **Building a commitly**
        * Version in the top level package.json: `1000.0.0-30e54adce`
        * This is the scenario our CI is facing on **every commit**.
        * Everything should be **built from source**. Artifacts should be available for a user that decides to use this nightly.
    * **Building a nightly**
        * Version in the top level package.json: `0.0.0-20220816-2027-543e11e5f`
        * This is the scenario our CI is facing **every night**.
        * Everything should be **built from source**. Artifacts should be available for a user that decides to use this nightly.
    * **Building a rc/stable**
        * Version in the top level package.json: `0.70.0-rc`.`0`
        * This is the scenario our CI is facing on the release branch
        * Everything should be **built from source**. Artifacts should be available for a user that decides to use this nightly.
        * **Plus:** Release tester on this scenario should be able to test this version of React Native, without having to build everything locally (i.e. by using the commitlies artifacts).
* **Users is using a numbered version of React Native** (nightly, RC or stable)
    * Version inside `node_modules/react-native` is: `0.70.0`
    * This is the scenario you’re facing if you `npx react-native init ...`
    * In this scenario, artifacts should be available for download, ideally with a stable URL which is parametric in the version of React Native. 

## Solutions

To address this problem, we should look into moving the binaries from the `react-native` NPM package to a different location. 

In this section we list the possible solutions.

### OCI Images on Github Container Registry

A [comment](https://github.com/react-native-community/discussions-and-proposals/pull/480#issuecomment-1206904964) from [Renaud Chaput](https://github.com/renchap) on the Monorepo RFC, raised this solution to our attention:

Github offers the possibility to use the [Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) to upload archives without size limits (for public repositories), as long as they conform to the [OCI (Open Container Initiative) specs](https://opencontainers.org/posts/announcements/2021-05-04-oci-dist-spec-v1/#:~:text=The%20OCI%20Distribution%20Spec%20defines,and%20pulling%20of%20container%20images.).  This is the same mechanism used by Brew to distribute binaries ([source](https://github.blog/2021-06-21-github-packages-container-registry-generally-available/#dont-just-take-our-word-for-it)).

Ideally we could create a container, which separate layer per artifact (i.e. one for Hermes iOS tarball, one for React Native Android, etc). Each layer will be a .tar.gz file we can freely populate.

#### Prototype

A prototype of this solution is available at [cortinico/space-unlimited-space](https://github.com/cortinico/space-unlimited-space).
The repo will showcase how to publish and retrieve arbitrary tarballs using HTTP without authentication.

#### Pro

* Elegant solution which relies on the Github infrastructure and doens’t involve external services (no need to auth, sign, or interact with new services).
* No limits to how much we can upload.
* Potentially we could access the Container Registry API to allow users to download only the layer they need.

#### Cons

* Need to write extra infrastructure to handle downloading the OCI images and placing the artifacts in the right folder.
* No caching mechanism out of the box (users with 2 React Native projects, will re-download everything). 
* URLs are hard to compose (e.g. https://ghcr.io/v2/cortinico/space-unlimited-space/anakin/blobs/sha256:57b8eee16897b0bb1d00190e99fb722dbaa32ecd1eb70686012ecf814ce329ba). We would need a mapping in place for knowing which blob contains what.
* Plain HTTP access is still not possible, we'll have to pass a `"Authorization: Bearer QQ=="` header, making it hard to consume via CocoaPods or other tool which are expecting HTTP urls ([see here for a curl request](https://github.com/cortinico/space-unlimited-space#consuming-via-http))

### Separate NPM Package

Ideally we could look into splitting `react-native` into smaller packages:

* `@react-native/android-prebuilts` will contain the `android/` folder
* `@react-native/hermesc-prebuilts` will contain the `sdks/hermes` folder

This will partially mitigate the issue as the `android-prebuilts` package will still hit the NPM limit once we include debug symbols. This will require further splitting such as:

* Per build-type split `android-debug-prebuilts` and `android-release-prebuilts`
* Per library split `react-native-prebuilts` and `hermes-engine-prebuilts`
* Per ABI split `x86_64-prebuilts`, `arm64-v8a-prebuilts` and so on.

#### Pro

* Uses a packaging system which is familiar for web developers
* We should be able to reuse most of the publishing pipeline we have in place at the moment

#### Cons

* We will be forced to “recompose” the various prebuilts from the several NPM packages to a Maven Local repository inside users’ `node_modules` folder.
* All the users will be downloading all the pre-builts for all the build-types/split/abi whenever they `yarn install`. Therefore users who develop for a single platform incur a download cost for binaries they will never use, vs delaying the download until the specific native build system needs the artifact.
* Having [github.com/facebook/react-native](http://github.com/facebook/react-native) as currently set up (not a proper monorepo) makes really hard to create new NPM packages.
* We will still be having a hard limit on NPM packages so we’ll have to handle further splitting as our library/SDK grows. This is especially more critical as we have more and more C++ code, therefore we’re generating more and more native pre-builts.

### Dedicated Maven Repository

Ideally we could look into publishing the `/android` folder into a remote Maven Repository (most likely Maven Central).
We could re-use a Maven Repository to store artifacts which are not Android-specific (e.g. the Hermes iOS Tarball).

#### Prototype

A prototype of how the storing of the Hermes iOS Tarball on Maven would look like is available on: [react-native#34812](https://github.com/facebook/react-native/pull/34812).

This has been tested against RN Tester iOS on `main` and the tarball download is working correctly. 

#### Pro

* For Android: Uses a packaging system which is familiar for Android developers and re-uses the already existing caching/perf improvement that are shipping with the platform pipeline.
    * A practical benefit of this is that users will have only a local copy of react-native or hermes-engine inside their `~/.gradle` folder instead of having multiple copies, one for every RN project.
* For iOS: We would have stable URLs which can be easily composed with the release version (e.g. https://repo1.maven.org/maven2/com/facebook/react/react-native-artifacts/0.0.1/react-native-artifacts-0.0.1-hermes-ios.tar.gz)
* We will end up using a single solution for both platforms, resulting in reduced maintainance costs.
* Users will download the artifact they need at build time only when needed
    * E.g. iOS-only users will benefit of faster `yarn install` and smaller `node_modules` folder.
    * Users which are not using Hermes will benefit similarly as well
* We won’t have any limits to the assets we upload and we can freely distribute debug symbols or different variants of our libraries.
* Companies using Maven Proxies (like Nexus) will benefit from this setup out of the box.
* Publishing GPG Signed artifacts will increase the trust users put on our distribution as they could verify that the artifacts haven’t been altered.
* This mechanism is the same used by Flipper and other libraries published by Meta (e.g. FBJNI) which we depends on.
* The React Gradle Plugin can be published on Gradle Portal, where all the other Gradle Plugin live.

#### Cons

* ~This solution addressed only the Android/JVM side of the problem~ See the prototype paragraph for a solution that works on both platforms.
* We will have to publish to remote Maven Repositories as part of our RN release process (i.e. a new step inside the CircleCI config, means yet another thing that can be broken) + we’ll have to maintain GPG keys for publishing.
* We’ll have to make sure NPM packages and Maven pubblications are syncronized with a version number.

### Relying on CircleCI Artifacts

Currently, several of our CircleCI jobs are producing and storing artifacts on CircleCI.
CircleCI artifacts have a [retention of 30 days](https://circleci.com/docs/persist-data#managing-network-and-storage-use), which can be optionally extended.

For nightlies and commitlies, we could setup our infrastructure to fetch artifacts directly from build jobs. CircleCI artifacts have stable URLs and can easily be downloaded without auth tokens.

#### Pro

* We reuse most of our storage current infrastructure. We just need to update the CircleCI config to store all the artifacts we need.

#### Cons

* We rely on CircleCI status to be up and running for being able to create and run react native apps.
* We need extra logic to properly configure react-native so that the Artifacts URL are populated correctly (i.e. how can a nightly know what’s the build Id of the job that generate it?)

### Hybrid Solutions

We could potentially apply separate solutions for different artifacts, to benefit from platform specific optimizations.

For example: 

* store hermesc binary, hermes iOS tarbal on a OCI image 
* store hermes android archive, react native android archive and Gradle Plugin on Maven Public repositories.

#### Pro

* We get the best of both worlds. Optimized caching and security of Maven, while still being able to distribute non maven artifacts.

#### Cons

* We need to authenticate and maintain the infra to interact with two distinct systems.

### Using Github Releases

We could extend on Github Releases as we’re doing today for the Hermes iOS tarball. That could work but has several side effects which are making this solution unpractical

#### Pro

* Simple solution, with stable URLs which are version specific

#### Cons

* Will create a lot of noise in the release space (i.e. we need to create a GIthub release for every nightly).
* Practically unusable for commitlies and release testing (as the release is ‘not there yet’).

## Implemented solution

After having investigated both the OCI and the Maven solution, we decided to move on with the **Dedicated Maven Repository** solution.

Starting from React Native 0.71.x we'll be shipping the following changes:

- We'll now distribute **prebuilts** for:
  - React Native Android - Coordinates `com.facebook.react:react-android`
    - Please note that the previous coordinates `com.facebook.react:react-native` have been voided due to [#35210](https://github.com/facebook/react-native/issues/35210)
  - Hermes Engine Android - Coordinates `com.facebook.react:hermes-android`
    - Please note that the previous coordinates `com.facebook.react:hermes-engine` have been voided due to [#35210](https://github.com/facebook/react-native/issues/35210)
  - Hermes Runtime for iOS - Coordinates `com.facebook.react:react-native-artifacts`

Prebuilts will be available on
- For **Stable** versions, on Maven Central ([browsable here](https://repo1.maven.org/maven2/com/facebook/react/))
- For **Nightly** versions,on Sonatype's Snapshot Repository ([browsable here](https://oss.sonatype.org/content/repositories/snapshots/com/facebook/react/))

This solution is flexible enough that will allow us to relocate our artifacts to other Maven repositories in the future, if needed.

The templates for both Android & iOS have been updated to consume the proper prebuilt from the correct URL.
