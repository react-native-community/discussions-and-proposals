# What repositories belong in React Native Community?

## Summary of the current state

#### The React Native Community organization has no official policy for what repositories belong or do not belong in the organization. It currently is home to four categories of repositories:

1) modules that were initially in the react-native repository but separated out (async-storage, etc).
2) packages used to support these modules (bob, etc).
3) resources for the react-native community (discussions-and-proposals, upgrade-helper, releases, etc).
4) arbitrary modules that were added at some point for different reasons (video, camera, linear-gradient, etc).

> :bulb: [Check out this gist](https://gist.github.com/brentvatne/9ab778c9d68f48e47c9987690232d131) for a full list of repositories and their groupings

#### React Native Community is perceived by developers in the ecosystem to be a 2nd party organization (not quite "core", blessed and worked on by Facebook and the core team, and not on the same level as other third parties) and packages are therefore assumed to be high quality, well supported, and generally trustworthy.

Perceived affiliation between Facebook and the repositories in the organization is not desirable for Facebook or the community, but it's hard to avoid. The organization has some "official" repositories like `discussions-and-proposals` and `releases`, and so putting other libraries next to them naturally has this effect. The result is that the libraries in the organization have a marketing advantage over other libraries in the ecosystem, and this is counterproductive to creating an ecosystem where the best projects rise to the top.

#### Developers use the React Native Community organization to discover packages ("does React Native Community have a package for X?") rather than a tool that is better suited for the job.

Given the previous point, it makes sense that users will make the React Native Community organization their first stop in discovering libraries to use in their projects. GitHub organizations are not meant to serve as a directory for a community of projects and does a poor job of it compared to a purpose-built tool, such as React Native Directory, which is capable of listing projects from any organization and providing additional context to help developers make an informed decision.

#### Authors and users of libraries frequently request that a library be moved to React Native Community.

This could be because it is no longer maintained, or the author wants help, or users want more confidence that it will continue to be maintained, or the author wants to promote their library and get a boost to their library downloads and improve their CV or ability to sell their services to clients by association with the organization.

The incentives for a developer to try to get a repository into the React Native Community organization hint at the underlying issues discussed above.

There is no clear benefit to including tens or hundreds of libraries maintained by distinct groups of people (or not maintained) in a single organization, and so we have been holding off on adding new repositories to the organization until some clarity is achieved (hopefully through this discussion).

## Proposed change

#### React Native Community will transition towards hosting only the repositories that fit into category 3.

The organization will contain repositories that act as forums for structured discussions and/or provide information about React Native and the ecosystem. These repositories may be purely textual, or they may be tools like the Upgrade Helper and React Native Directory, depending on what is better suited for the job. The organization will not include libraries that developers install and use in their projects.

Libraries that fit into categories 1, 2, and 4 will be moved to other organizations or users. For example, Callstack might take over some projects that they actively maintain and Microsoft might take others. Unaffiliated developers would just move the repository to their own personal GitHub or an organization built specifically for the repository, eg: http://github.com/react-navigation/. An action plan can be detailed at a later date.

This also solves the issue of arbitrarily blessing specific solutions that may not be the best now or in the long term for the ecosystem. Rather than blessing specific libraries, we can direct developers to the [React Native Directory website](https://reactnative.directory) and continue iterating on it to help developers assess quality and discover and select the best packages for their needs. Additionally, we can move React Native Directory back to React Native Community. Developers who wanted to add their repository to the organization for marketing purposes can put it on the directory instead.

### Tradeoffs

- Many packages have moved their name to the @react-native-community npm scope and removing that would create thrash. This is not a big deal for packages that are directly consumed by the developer, in my opinion, but it can be in places where the package is heavily depended on by other packages, eg: @react-native-community/async-storage is a building block for many other libraries.
- Lean core libraries include some fundamental building blocks for most apps, like slider, net-info, progress-view, webview, and async-storage. Will moving them out of the organization make them more difficult to discover?

## Result

Partners voted unanimously to approve the proposed change on 08/19/2020.