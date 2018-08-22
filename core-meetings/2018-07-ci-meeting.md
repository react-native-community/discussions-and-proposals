# Better CI, Testing, and Proposals: July 2018 Core Meeting

In case you missed [the release of 0.56](http://facebook.github.io/react-native/blog/2018/07/04/releasing-react-native-056), the teams behind React Native have been busy! Nevertheless, as stewards of React Native engineers worldwide, we've contemplated ways to improve alignment and visibility of the framework's direction, reason why we decided to bring back the monthly meetings (with an updated format), where community members and Facebook alike joined to discuss issues and propose responses.

The community team is made up of volunteers who help maintain React Native, contribute to education materials, and improve the React Native developer's environment.

## The core discussion

This month, the group chose to discuss React Native's continuous integration tooling.

The React Native repository has a complex [continuous integration suite](https://circleci.com/gh/facebook/react-native) that automatically tests pull requests and master to help guarantee quality. These tests are actually split into five separate suites, ranging from checking for Flow errors to iOS end to end testing. CI should give confidence that things are good when merging pull requests and making release candidates. Recently though, CI success has been a challenge. It's even caused slowdowns in making releases.

## Make CI stay green

In our discussion, the group identified two main problems with our use of CI today: automated test coverage is insufficient and Facebook inadvertently syncs problem changes.

### Better test coverage

While the goal of CI is to help gain confidence in merging PRs, many areas of the codebase don't have enough test coverage to offer that. For instance in the first stable release of 0.56, there was an issue using the CLI on Windows. There are many areas of the codebase today that lack robust tests in the Github repository.

One of the proposals is to offer faster landing of pull requests that include automated test coverage.

### Don't let master languish

As mentioned in [How to Contribute](https://facebook.github.io/react-native/docs/contributing#our-development-process), the Github repository mirrors an internal repository used at Facebook. Master commits are synced between the two by an automated process, but there are some differences between the two codebases (e.g. Facebook doesn't use Cocoapods or the CLI internally). Another difference is in CI: the internal repository uses Phabricator and the open source repository uses CircleCI. Due to factors beyond the React Native team, the CIs used by the community team and the Facebook team may not get to be exactly similar.

One of the proposals is to sync changes into a branch rather than directly into master. This is currently being drafted.

Another proposal is to file tickets internally at Facebook when one of their changes breaks the community CI. Hector is currently working on a bot for this.

Sometimes though, the trouble syncs are actually in React Native's dependencies like Yoga, Folly, and Metro. Facebook internally does not use version releases; they consider master to be the current version at all times. Sometimes changes to these break the CI, and that can be difficult to troubleshoot. A proposal was made to move all of these dependencies into the React Native repository, and a counter proposal was made to have the community maintain package releases of the dependencies.

## Increasing awareness

Community engagement was a recurring topic on the call. Most of the discussion focused on automation, but what about increasing knowledge transfer? There's a proposal on teaching about the "under the hood" parts powering React Native, creating a separate recurring monthly tech talk.

### Proposals

All of these proposals need a home, so we're creating a location and format for them. Our ambition is to bring these conversations into the public but not force a rigid process. We're working on this now.

---

This month's topic was CI, but we are planning to cover much more. Expect to see many of these proposals soon in this repository, where we hope you will help us make React Native even better.
