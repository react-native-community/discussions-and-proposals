---
title: React Native as a monorepo
author:
  - Lorenzo Sciandra
  - Tommy Nguyen
  - Nicola Corti
date: 2022-05-18
---

# RFC0006: React Native as a monorepo

## Summary

This proposal is not much as in introducing something completely new, but more to discuss a solution to an existing problem. To put simply, the current shape of the `react-native` codebase hosted in GitHub is lacking structure and consistency and this is leading to a number of issues (detailed below).

This proposal is about re-shaping the existing codebase on GitHub into what is commonly expected of a monorepo in the JavaScript ecosystem (see, for example, [Babel.js](https://github.com/babel/babel) or [Next.js](https://github.com/vercel/next.js)): consistent versioning, naming and tooling.

### Table of Content

- [Motivation](#motivation)
  - [A IRL scenario of this being a problem: `react-native-codegen` and RN 68](#a-irl-scenario-of-this-being-a-problem-react-native-codegen-and-rn-68)
- [Detailed design](#detailed-design)
  - [The proposal](#the-proposal)
  - [Going the extra mile: Hermes](#going-the-extra-mile-hermes)
  - [Implementation detail considerations](#implementation-detail-considerations)
  - [Other considerations](#other-considerations)
- [Plan of action](#plan-of-action)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [How we teach this](#how-we-teach-this)
- [Future work](#future-work)
- [Unresolved questions](#unresolved-questions)

## Motivation

To quickly explain the motivation behind this proposal, here's the current shape of the React Native codebase hosted on GitHub and corrispective npm packages:

![current status of the react-native GitHub codebase](/proposals/assets/0006-current-status.png "Current Status")

It should quickly become apparent that there are a few discrepancies:

- naming convention is not consistent (ex. `@react-native/assets`, `react-native-codegen`, `@react-native-community/eslint-plugin`)
- semver versioning is not consistent (certain packages are on a major version such as `@react-native/assets`, others are on a patch level such as `@react-native/babel-plugin-codegen`)
- semver is not aligned or coordinated with the `react-native` package semver
- no changelogs for any of these other packages

On top of this, there are some more substantial issues:

- releasing a new version of any package that is not the main `react-native` one requires an engineer manually releasing it via CLI on a local machine (which is potentially also a security issue, given how some of these packages are owned by members of the community)

  - which also means that the same person then needs to manually bump the dependency in other parts of the codebase, since they are fixed. For example, see how root level package.json depends on `"@react-native/polyfills": "2.0.0",` ([permalink](https://github.com/facebook/react-native/blob/b331229b06dde31ff92091a163cfabb030082925/package.json#L108)).
  - this also affects the nightlies cycle: the only package that gets nightly releases is `react-native`, and no other packages (which implies a high risk of the nightly being broken because the rest of the toolchain not having a corrispective release).

- because of the root `package.json` being not just the `react-native` package but also the root/CI for the repo, during the release script for `react-native` there's some 🪄magic🪄 that happens to copy over some dependencies from `repo-config/package.json` into the root one (see [this commit](https://github.com/facebook/react-native/commit/6efb51777c620a95e457488fbc7df5c272b3c695#diff-7ae45ad102eab3b6d7e7896acd08c427a9b25b346470d7bc6507b6481575d519)). It's a weird pattern that is undocumented and I'm scared of the issues it creates, since (as all the others) `repo-config/package.json` is manually maintained.

  - for example, 0.69.0-rc0 was still depending on React 17 in the 0.69 branch; it was addressed afterwards with [this commit](https://github.com/facebook/react-native/commit/a862d9294bf22ba2304fe5d742010dc210a2ef5a)

- because this configuration is mostly "github repo only", there's an high risk of things getting outdated or changes slipping between the cracks (such as the `repo-config` example mentioned there), and only being addressed after someone hits them as a third party consumer.

- The current setup is not encouraging modularization (that makes situations like the [RFC about ios and android packages](https://github.com/react-native-community/discussions-and-proposals/pull/49) harder). Essentially creating a new `@react-native/*` NPM package today is over-complicated and the infrastructure is not able to handle this properly.

- This setup makes it hard to have consistent testing in the CI:
  - there are two different build patterns for [RNTester](https://github.com/facebook/react-native/tree/main/packages/rn-tester) and for [the template](https://github.com/facebook/react-native/tree/main/template).
  - template has to do 🪄magic tricks🪄 to simulate an npm package for react-native when testing - [see more here](https://github.com/facebook/react-native/blob/main/scripts/test-manual-e2e.sh#L109).

### A IRL scenario of this being a problem: `react-native-codegen` and RN 68

To clarify further how this current structure negatively affects maintaining the codebase, here's a quick example that has happened relatively recently.

`react-native-codegen` is one of the packages shown in the graph as one that is on a `0.0.X` versioning. This meant that with RN `0.68.0`, the version of `react-native-codegen` it depended on was the one the codebase was at the time of cutting the `0.68-stable` branch (`0.0.13`).

After releasing `0.68.0`, it was quickly realised that there was an issue with this version (details are unnecessary) that required a fix; this fix also needed to affect code within the `react-native-codegen` folder.

Here's the catch: in the time between the cut of `0.68-stable` branch and the release of `0.68.0`, work on the main branch kept on going and by when the fix was needed, `react-native-codegen` was on `0.0.16`. This meant that we needed to apply a 0.68 specific fix on `react-native-codegen` in its code shape in the `0.68-stable` branch, and do a release in such a way that it would only get picked up by RN 68. But because of the way semver works on Node, the only version we could release was `0.0.17` (`0.0.13.1` is not a thing). And it should be clear that releasing an higher version number of a package, that contains older code, is a very bad situation.

To address this, in the end with the releases team we had to coordinate a triple version number change for `react-native-codegen` so that it would reshape into:

- `0.68-stable` uses `0.0.17`
- `0.69-stable` uses `0.69.x`
- `main` uses `0.70.x`

And all of this had to be done manually, and after each version bump for this package, a new version of the 0.68 and 0.69 had to be released with the updated dependency on the new appropriate version.

This proposal aims to address this problem so that it won't have to be dealt with in the future.

## Detailed design

### The proposal

This proposal will seem deceptively simple: to decouple the root level of the package from being both root and `react-native`, to just be the former, and move `react-native` as a package in the `packages/` folder.

The graph here shows in a bit more detail the end goal, along with a few more details and side-changes that are needed to fully address the current situation:

![proposal for the react-native GitHub codebase](/proposals/assets/0006-proposal.png "Proposal")

Reiterating the changes proposed in the graph:

- we need to move the `react-native` specific code in its own folder within the `packages/` folder. This is the most error prone step to handle, as many paths and variables might have to be modified (for ex. in the CI configurations) to accommodate for this change.
- we have to **rename** some of other packages as shown above, and in this table:

  <details>
  <summary>Expand table of package.json/npm names</summary>

  | old name                              | new name                           |
  | ------------------------------------- | ---------------------------------- |
  | @react-native/assets                  | @react-native/packager-assets      |
  | @react-native/babel-plugin-codegen    | @react-native/babel-plugin-codegen |
  | @react-native-community/eslint-config | @react-native/eslint-config        |
  | @react-native-community/eslint-plugin | @react-native/eslint-plugin        |
  | @react-native/normalize-color         | @react-native/normalize-colors     |
  | @react-native/polyfills               | @react-native/js-polyfills         |
  | react-native-codegen                  | @react-native/codegen              |
  | react-native-gradle-plugin            | @react-native/gradle-plugin        |

  </details>

  <details>
  <summary>Optional: change name of folders</summary>

  | old path                                                   | new path                                   |
  | ---------------------------------------------------------- | ------------------------------------------ |
  | react-native/packages/assets                               | react-native/packages/packager-assets      |
  | react-native/packages/babel-plugin-codegen                 | react-native/packages/babel-plugin-codegen |
  | react-native/packages/eslint-config-react-native-community | react-native/packages/eslint-config        |
  | react-native/packages/eslint-plugin-react-native-community | react-native/packages/eslint-plugin        |
  | react-native/packages/normalize-color                      | react-native/packages/normalize-colors     |
  | react-native/packages/polyfills                            | react-native/packages/js-polyfills         |
  | react-native/packages/react-native-codegen                 | react-native/packages/codegen              |
  | react-native/packages/react-native-gradle-plugin           | react-native/packages/gradle-plugin        |
  | react-native/packages/rn-tester                            | react-native/packages/tester               |

  </details>

  - _(this step is needed to allow to reset the versioning and introduce alignment across the board)_

- we will need to release a new version of the packages with the new npm name/org (and version number)
  - this will require a ripple effect of renaming also the places in which those packages are consumed, and to the new version, in the rest of the codebase.
- we want to introduce tooling to avoid having to manually release and version packages.
  - in particular, we recommend adding the tool [`changeset`](https://github.com/changesets/changesets) to the codebase to handle the release coordination (as in, once 0.70 gets created, all the packages in that branch get versioned to `0.70.x`) and bump packages accordingly when new versions are needed
  - as an alternative, another one of Meta's projects in OSS, [Metro](https://github.com/facebook/metro) could be used as an inspiration for how to setup this scenario: it addresses the naming and versioning in the same manner as described here, and it's a project that has a lot of similarities to RN in the fact that it's also a "partial replica of monorepo code" scenario. It uses [Lerna](https://lerna.js.org/).

In closing this section, we also want to acknowledge how this proposal is deliberately not introducing any high degree of automation or advanced tooling - this is because we are well aware that this repository is but a "partial mirror" of how the react-native code is shaped within the Meta monorepo, and adding more extensive and invasive tooling would require also introducing them to that monorepo. So we opted for the minimal footprint that would be OSS-side only (with the tradeoff of more custom, local code and scripts).

### Going the extra mile: Hermes

An element that is not presented in the graph above is Hermes engine as a package in this repo (ex. `@react-native/hermes-engine`). This is intentional as it's a more complicated subject, but it's interesting to mention, given the recent decision to move Hermes to a "build from source" scenario ([details](https://github.com/reactwg/react-native-new-architecture/discussions/4#discussioncomment-2382960)).

An interesting option could be to have it yes as a subfolder (via [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules), maybe?) and handle it as all the other packages above (even for versioning & publishing).

Another option, that would build on top of the (new) existing process, would still rely on git tags on the `facebook/hermes-engine` repo, but would basically "pull that code" within a `<root>/packages/hermes-engine` folder and handle the release process of Hermes (and its build process) inside it, and publish the artifacts & pre-build separately from the main `react-native` node_module. This would help alleviate the problem of the main `react-native` module being too heavy, and re-align Hermes' versioning to the `react-native` one.

### Implementation detail considerations

- Some parts of this work will have to start from Meta's side (see [comments in the other proposal](https://github.com/react-native-community/discussions-and-proposals/pull/49/files#r255135557)): there's a lot of company specific knowledge involved in how the `react-native` repo gets created for GitHub, and in how the code is shaped in the internal monorepo. Also, renaming and pathing will affect the interal codebase so coordination will be needed.
- Since Metro by default doesn't handle symlinks well (see [this issue for details](https://github.com/facebook/metro/issues/1)), it could be that for RN Tester to keep working we might have to add [`@rnx-kit/metro-resolver-symlinks`](https://github.com/microsoft/rnx-kit/tree/main/packages/metro-resolver-symlinks)
- the changelog generator ([`@rnx-kit/rn-changelog-generator`](https://github.com/microsoft/rnx-kit/tree/main/incubator/rn-changelog-generator)) will need to be worked on to accommodate generating changelogs for any/all/some of the packages in the new monorepo structure
- the logic for the release of nightlies will also need to be revisited to account for the new structure (it would be interesting to star evaluating for which packages a new nightly would be needed at which point); to be clear, right now the existing logic only works for the core `react-native` package.

### Other considerations

- we could consired renaming `react-native` into `@react-native/core` but no strong opinion; given how this is the "front-facing" package it might be easier and more digestable for consumers of react-native to just keep stay on the current convention.
- given the many artifacts involved in the react native toolchain for building an app, there's also the need to figure out a more stable and "automatable" via for creating and distributing these assets. For that purpose, a [separate RFC has been set up here](https://github.com/react-native-community/discussions-and-proposals/pull/508).

## Plan of action

After several brainstorming, we envision the work split in those phases:

_sidenote: all work in the main repo around this effort will be tracked by the label ["Tech: Monorepo"](https://github.com/facebook/react-native/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3A%22Tech%3A+Monorepo%22)._

### Phase 1 (Package Renaming & Re-versioning)

Package renaming & versions alignment. We are going to track this via an umbrella issue:
https://github.com/facebook/react-native/issues/34692

As a rule of thumb, we decide to **not** do folder renaming: it brings little value, at least at this early stage.

Ideally we should be done with all the renaming by 0.71.x so that we can annonce that this change has happened in the release notes. We don't expect any user facing impact as users should depend only on `react-native`. Still it's worth to make sure that all the renaming/bumps are bundled together in a single version.

### Phase 2 (Automation)

Once all the versions are aligned, we can work on automation to support:

- Publishing of all the packages from the CI
- Automating the bumping of the version

### Phase 3 (Moving the `react-native` package)

Once we're done with the setup, we can look into moving the top level package inside `packages/react-native` and have a proper yarn workspace in the top level `package.json` (by also removing the `repo-config/` folder).

### Phase 4 (Cleanup)

Once we're done with the full monorepo setup, we'll into follow-up tasks such as:

- Deprecating older packages on NPM (those that have been renamed)
- Remaing folders, for those we believe we'll get value out of the rename.

## Drawbacks

- Old open pull requests will likely need to be rebased and re-arranged
- Some of the native build scripts (Gradle, BUCK, Xcode, etc.) and CI scripts and config will need to be updated for the new code locations

- _(Adding this bullet point to easily allow comments on other drawbacks that might have not been considered)_

## Alternatives

Since this is basically a fix and not a re-invention, alternatives have not been really explored deeply. Most of the alternative takes boil down to tech choices details in ex. which tool to use to handle the monorepo setup, and not really in the final overall shape of the codebase.

## Adoption strategy

This workstream will span multiple minor releases (see the various steps), so we will need to communicate that each user-facing change as a step to upgrade to the new version.

Tools like [`dep-check`](https://github.com/microsoft/rnx-kit/tree/main/packages/dep-check) or a codemod should be enough to support edge-cases in which some packages were imported directly.

No breaking changes are expected, it's just a matter of naming & semver convention re-alignment.

## How we teach this

Nothing to teach, we just need to ensure that the first version to be released after this will have the documentation and blogpost communicate accordingly the new package names (but since `react-native` will stay `react-native`, to many folks this will be transparent).

Also, the documentation about handling releases will have to be slightly modified if new scripts will get added.

## Future work

We just wanted to point out how this is basically a big refactoring to enable more complex and impactful changes to be more easily done in the subsequent future, such as the integration with Hermes mentioned above or the other proposal ["Move iOS and Android Specific Code to Their Own Packages"](https://github.com/react-native-community/discussions-and-proposals/pull/49). In fact, this proposal addresses [the latest comment](https://github.com/react-native-community/discussions-and-proposals/pull/49#issuecomment-1046073630) on that RFC from [@cortinico](https://github.com/cortinico), in particular resolves points 1 and 2.

## Unresolved questions

_feel free to add something via comments_
