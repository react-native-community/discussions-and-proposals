# Open Source Maintenance: November 2018 Core Meeting

As stewards of React Native engineers worldwide, we've contemplated ways to improve alignment and visibility of the framework's direction, where community members and Facebook alike joined to discuss issues and propose responses.

The community team is made up of volunteers who help maintain React Native, contribute to education materials, and improve the React Native developer's environment.

## The core discussion

This time, many members from both the community and the FB team met to talk about the Open Source repository and its maintenance.

The first part of the conversation was about two aspects of the current release (0.57.x):

- how the React sync commits land in master, and have impacted the releases
- how to tackle the patch releases and cherry picks request, given that over time this is more complex and prone to merge conflicts (because of the speed at which master branch moves)

During the discussion, the FB team also clarified a couple of points from their latest blogpost about [Open Source Roadmap](http://facebook.github.io/react-native/blog/2018/11/01/oss-roadmap), here reported for transparency sake:

- the internal FB team is _not_ switching React Native's source of truth to be GitHub first: they are simply changing _how_ they consume the React Native APIs internally. Which means that, internally, instead of writing `const View = require('View')` they will be writing `const {View} = require('react-native')`

  The GitHub repository will continue serving as a mirror of the commits that land in the internal react-native-github directory in Facebook's monorepo, and pulling in changes from open source will still require importing a PR to the monorepo rather than merging directly to master on GitHub.

- the FB team is not foreseeing taking management or ownership of the release cycle\*; this was motivated by a few reasons, such as:
  - Wanting to generate more community participation.
  - The community probably know more about this than FB: given that the releases is how the community consumes RN, they should have the power to decide the release cycle.

The second and last part of the meeting was dedicated to the upcoming version 0.58 - it was clarified that the JSI that has landed on master should not directly affect developers consuming React Native, and an issue to facilitate the discussion about it will be open on this repo (but no blog post should be expected).

During the meeting, the subject of having a single organization to host the React Native components & libraries maintained by the community was briefly touched (given that currently there are two main ones, React Community and React Native Community) - but there wasn't time left so this will be discussed in a future meeting.

### Actions that will be taken

1. In patch level releases, from now on, there won't be important dependencies upgrades (unless strictly necessary)
2. A few tools to automate (at least partially) the release cycle will be tested and added to the process to reduce the workload on the maintainers (you can contribute to this conversation [here](https://github.com/react-native-community/discussions-and-proposals/issues/53))
3. The [Releases.md](https://github.com/facebook/react-native/blob/master/Releases.md) file will be updated to reflect the current set of steps necessary for a release
4. For cherry pick requests, in case of complexities emerging during the release process, the author of the request will be asked to create a PR towards the branch
5. The FB team will attempt to be more empathetic when landing commits like React syncs to prevent issues
6. Another maintenance meeting will happen before end of the year, and among other things we'll discuss about a shared guideline for how to maintain repos owned by the community in order to help each other

---

This time's topic was Open Source Maintenance, but we are planning to cover much more. You can check both the issues and the PRs of this repository to see what conversations are ongoing, and we hope you will join us in making React Native even better!

\* this conversation sparked more discussion on the larger topic of maintainability & sustainability of the RN project - that is still ongoing. You can see the key points and join the conversation in this [dedicated discussion](https://github.com/react-native-community/discussions-and-proposals/issues/58).
