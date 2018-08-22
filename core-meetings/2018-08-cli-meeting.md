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

Which also means that potentially could make React Native more "packager agnostic".

That said, this is not "mandatory" for having first class support of third-party libraries such as RN-Windows or RN-Dom.

---

This month's topic was CLI, but we are planning to cover much more. You can check both the issues and the PRs of this repository to see what conversations are ongoing, and we hope you will join us in making React Native even better!
