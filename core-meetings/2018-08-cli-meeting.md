# The CLI and its future: August 2018 Core Meeting

As stewards of React Native engineers worldwide, we've contemplated ways to improve alignment and visibility of the framework's direction, where community members and Facebook alike joined to discuss issues and propose responses.

The community team is made up of volunteers who help maintain React Native, contribute to education materials, and improve the React Native developer's environment.

## The core discussion

This month, the group chose to discuss React Native's CLI tooling and its future.

This conversation is part of the greater effort to [slim down the core repository](https://github.com/react-native-community/discussions-and-proposals/issues/6), while ensuring the code health. In particular, the starting point of this meeting was the [proposal](https://github.com/react-native-community/discussions-and-proposals/pull/13) of moving the CLI out of the main repo by [@grabbou](https://github.com/grabbou).

## Better decoupling

In our discussion, the group identified that by moving the CLI tooling outside of the main repo, the Metro dependency would be _potentially_ decoupled from the main react-native library - it seems that the only files outside `/local-cli` referring to Metro are:

- [HMRClient](https://github.com/facebook/react-native/blob/master/Libraries/Utilities/HMRClient.js)
- [jest/preprocessor](https://github.com/facebook/react-native/blob/master/jest/preprocessor.js)

Which also means that potentially moving it outside could make React Native more "packager agnostic".

That said, extracting it is not "mandatory" for having first class support of third-party libraries such as RN-Windows or RN-Dom.

A potential issue that may raise from this complete decoupling would be related to making sure that the different repos are always up-to-date with the newer versions of Metro (which are released quite frequently).

## `upgrade` and `react-native-git-upgrade`

The conversation then shifted towards a portion of the React Native DX which is related to updating existing React Native projects to newer versions; in the CLI there are two different commands to help with it, but newer releases are showing their limitations.

We are all aware of the complexities that updating to a newer React Native version can cause, and given that `react-native-git-upgrade` is the "most user-friendly" approach - but at the same time the least maintained - we tried to get back to the reason why it was merged and how it works: we found the [original PR](https://github.com/facebook/react-native/pull/11052) and the [blog-post](https://facebook.github.io/react-native/blog/2016/12/05/easier-upgrades) to help us in this.

Given that most of the group actually manually upgrades the files to ensure a smooth transition in their projects, we discussed a lot about what an alternative to `git-upgrade` would look like. Since we couldn't come up with a "one for all" solution (we even considered code-mods) we agreed on:

1. trying to get more members of the community involved with maintaining this portion of the CLI.
1. similarly to Expo's approach, to write clear "how to manually upgrade" steps in the Changelog for new versions

---

This month's topic was CLI, but we are planning to cover much more. You can check both the issues and the PRs of this repository to see what conversations are ongoing, and we hope you will join us in making React Native even better!
