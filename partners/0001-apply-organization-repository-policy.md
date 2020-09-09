# Plan for applying the new React Native Community repository policy

To summarize the descision made in ["What repositories belong in React Native Community?"](https://github.com/react-native-community/discussions-and-proposals/blob/master/partners/0000-organization-repository-policy.md):

> The organization will contain repositories that act as forums for structured discussions and/or provide information about React Native and the ecosystem. These repositories may be purely textual, or they may be tools like the Upgrade Helper and React Native Directory, depending on what is better suited for the job. The only exception to this is React Native CLI, which will continue to live in React Native Community as long as the maintainers believe this to be the best home for it.

[The partners making this decision](https://github.com/facebook/react-native/blob/master/ECOSYSTEM.md#partners) prioritize the health and growth of the community of React Native developers, which is much larger than those with maintainer access to any specific organization. High quality modules can and should come from anyone, anywhere. We recognize that developers value using the react-native-community brand as a library quality heuristic, but we believe that there are heuristics that are more useful and also healthier for the community. Our focus will be on things that scale; helping discoverability, making it easy to find quality packages, and building tooling to help create and maintain these packages.

Now that a policy has been decided, we can move forward with enacting the changes to align the organization with it. As a maintainer of one of these packages, you don't need to wait to start making these changes. While you may follow what you think is best for your package, we are providing some recommended steps we think will help ease the migration.

That said, **we know that changing package names can be challenging and a difficult transition for the ecosystem**. We will take care to support the community through this migration as best as we can.

## Where on GitHub will the repositories be moved to?

Our recommendation for lean core libraries and libraries with multiple maintainers is to move them to their own dedicated organization, for example: react-native-webview would be moved to a new organization called react-native-webview, and would live at react-native-webview/webview. We have gone ahead and reserved all of the organizations by these names where possible, reach out to @brentvatne to have him transfer ownership to you if needed.

If a library has just one maintainer, or is maintained only by developers from a single company, then the repository may be better suited for the individual or company GitHub account.

Libraries that have been archived or otherwise abandoned will be moved to a React Native Community Archive organization in order to keep the repository list focused.

If you a maintainer of a repository in the React Native Community organization and have any questions or concerns about how to proceed, or if you lack the required organization permissions, please let it be known in #community-hub. If you are not sure if you have the right/authority to act on moving a repository but consider yourself to be the primary maintainer, please also raise this in #community-hub.

## What should be done about packages that are scoped under @react-native-community on npm?

We recommend deprecating the latest release with a message that the package name has changed, and republishing that exact same release under the new package name. Ideal package names may not always be available, but you might be able to reach out to the current owners of those packages and have them transfer ownership.

Most packages that were not formerly available as part of React Native core are already unscoped. If your package used to exist without the @react-native-community scope and you have since migrated to it, you may consider migrating back to the original package name.

## How will developers discover commonly needed packages like async-storage, slider, webview, net-info, and others?

[React Native Directory](https://reactnative.directory/) is the recommended resource for library discovery. We have surfaced it in the official documentation and may at some point merge the two. Further discussion on how we can surface some extremely commonly required dependencies should continue on [this issue](https://github.com/react-native-directory/website/issues/444).