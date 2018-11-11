# Open Source Maintenance: November 2018 Core Meeting

As stewards of React Native engineers worldwide, we've contemplated ways to improve alignment and visibility of the framework's direction, where community members and Facebook alike joined to discuss issues and propose responses.

The community team is made up of volunteers who help maintain React Native, contribute to education materials, and improve the React Native developer's environment.

## The core discussion

This time, many members from both the community and the FB team met to talk about the Open Source repository and its maintenance.

The first part of the conversation was about two aspects of the current release (0.57.x):

- how the React sync commits land in master, and have impacted the releases
- how to tackle the patch releases and cherry picks request, given that over time this is more complex and prone to merge conflicts (because of the speed at which master branch moves)

During the discussion, the FB team also clarified a couple of points from their latest blogpost about [Open Source Roadmap](http://facebook.github.io/react-native/blog/2018/11/01/oss-roadmap), here reported for transparency sake:

- the internal FB team is _not_ migrating to React Native Open Source, they are simply changing _how_ they consume the React Native APIs. Which means that, internally, instead of writing `const View = require('View')` they will be writing `const {View} = require('react-native')`

- the FB team is not foreseeing taking management or ownership of the release cycle, commending the community for how they have handled it so far

The second and last part of the meeting was dedicated to the upcoming version 0.58 - it was clarified that the JSI that has landed on master should not directly affect developers consuming React Native, and an issue to facilitate the discussion about it will be open on this repo (but no blogpost should be expected).

### Actions that will be taken

1. in patch level new releases, there won't be important dependencies upgrades (unless strictly necessary)
2. for cherry pick requests, in case of complexities emerging during the release process, the author of the request will be asked to create a PR towards the branch
3. the FB team will attempt to be more empathetic when landing commits like React syncs to prevent issues
4. another maintenance meeting will happen before end of the year, and among other things we'll discuss about merging the two GitHub orgs React Community and React Native Community and we'll create a shared guideline for how to maintain repos in them

---

This time's topic was Open Source Maintenance, but we are planning to cover much more. You can check both the issues and the PRs of this repository to see what conversations are ongoing, and we hope you will join us in making React Native even better!
