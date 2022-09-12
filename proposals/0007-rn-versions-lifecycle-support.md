# RFC: React Native Versions Lifecycle & Support



>As we look for ways to better support the React Native community, we will move towards supporting a limited number of versions of React Native better.


The React Native Core team is working to strike a balance between the speed of innovation with the amount of regular change to the platform. We understand that each release brings new value and has an adoption cost to the community.
This document describes how we intend to formally balance these concerns and provide clear and consistent support over time.
React Native team will work on best-effort basis to release a new version of React Native every four months. And will focus on speeding up the release cycle and put considerable effort into ensuring a seamless upgrade path.
This document describes the level of support we can provide and which set of versions is supported.

## Glossary

Below we are defining terms used in this document:

* **stable** **version** - Any version that doesn’t have a -alpha, -beta, -RC postfix in the version name.
* **latest version** - **** The latest stable version with the highest version number (e.g. 0.68.2).
    * The highest version number is defined following the [NPM server algorithm](https://github.com/npm/node-semver).
* **next version** - The next version that hasn’t been fully released yet, and is currently in development/testing (e.g. 0.69.0-RC1).
* **minor series** - A collection of versions (either stable or not stable) sharing the same minor version (e.g. **0.68**.0-RC0, **0.68**.0, **0.68**.1 are all part of the **0.68** minor series).

## Which versions are currently supported?

We’re supporting the **latest** **version**, and the latest versions from the **two previous minor series.**
We’ll also support the **next version** being developed, which will become stable after its release.

Versions:


|Version	|Type	|Status	|
|---	|---	|---	|
|0.70.0-RC2	|Release candidate	|Will be in support when stabilized	|
|0.69.4	|Latest stable	|In support	|
|0.68.3	|Previous minor series	|In support	|
|0.67.4	|Previous minor series	|In support	|
|<=0.66.3	|Old minor series	|Unsupported	|

## What is the level of support?

Due to support bandwidth, the React Native team, with the community's help, is looking into issues & PRs opened against one of the supported versions.

Issues & PRs opened against older versions would be considered only in exceptional cases. Please update your application to one of the supported versions and raise the Issue/PR targeted to that version.

Issues should contain a [**reproducer**](https://stackoverflow.com/help/minimal-reproducible-example) project regardless of which version they are targeting, for them to be considered.
Issues without a reproducer will require more effort to understand and fix, and are less likely to receive attention.

At this point in time, we are prioritizing issues that are related to:

* Latest version of React Native and two previous minor series.
* Use of the New Architecture
* Use of the Hermes Engine

## Cherry-Pick Requests

We’re **accepting cherry-pick requests** for the currently supported versions. A cherry-pick request is the request to include a fix in one of the supported versions and release a new point release with it.

Cherry-Pick requests should be submitted via the [React Native Releases Working Group](https://github.com/reactwg/react-native-releases/discussions/categories/patches), in the corresponding discussion thread.

Please note that each cherry-pick request will be assessed and approved individually. Cherry-Pick requests against unsupported versions will be rejected unless they involve security issues.

## Security Issues

Security issues should follow our issue report policy [described here](https://github.com/facebook/react-native/blob/main/CONTRIBUTING.md#security-bugs)


## Licence

React Native is open-source software distributed under the **MIT license**. Software is provided "as is" without warranty.
