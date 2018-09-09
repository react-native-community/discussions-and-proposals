# Metro configurations changes: September 2018 Core Meeting

As stewards of React Native engineers worldwide, we've contemplated ways to improve alignment and visibility of the framework's direction, where community members and Facebook alike joined to discuss issues and propose responses.

The community team is made up of volunteers who help maintain React Native, contribute to education materials, and improve the React Native developer's environment.

## The core discussion

This time, the core team - thanks to the fact that some of us were at React Native EU 2018 - met briefly to discuss some upcoming changes to the Metro configuration that will happen in future versions of the packager.

In particular, the two main changes that will be implemented are:

- [Inline requests](https://facebook.github.io/react-native/docs/0.56/performance#unbundling-inline-requires)
- [Ram bundling](https://facebook.github.io/metro/docs/en/bundling#indexed-ram-bundle)

These features are powerful advancements towards a more performant bundling and startup time for all React Native apps, but at the same time are "breaking changes" in the way they generate the bundle.

This means that just sliding these features in and turn them on by default is not a viable solution for many developers in the community, that would all of a sudden face a new change to handle.

Moreover, a new "format" of the bundle would interfere with the standard behavior of most _Over The Air_ update systems like Expo's or Codepush - since they by default generate non-ram bundled files to serve to the RN apps.

The last point discussed was about where the configuration for these feature should be possible to be written - newer versions of Metro can consume a `metro.config.js` file, but historically there is also a `rn-cli.config.js` to consider (that will not be deprecated for the forseeable future); another option was, similar to `jest`, to be able to have it inline in the `package.json`.

The conversation led to the following:

### Actions that will be taken

1. starting v0.58, all React Native projects will have these flags explicitly set to `false` by default (0.57 is not affected by this), since `react-native-git-upgrade` will make them appear in projects being updated. This will allow for easy optional "jump in" by developers.
2. in the Changelog for 0.58 we will write about these new flags
3. we will communicate about these changes to OTA services like Expo's and Codepush so that they will be able to create bundles accordingly to the new configuration
4. based on the future of the CLI, we'll try to have `metro` being configurable via `rn-cli.config.js` or similar files (and not only from `metro.config.js`)

---

This time's topic was Metro configuration, but we are planning to cover much more. You can check both the issues and the PRs of this repository to see what conversations are ongoing, and we hope you will join us in making React Native even better!
