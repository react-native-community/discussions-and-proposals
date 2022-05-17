---
title: React Native as a monorepo
author:
  - Lorenzo Sciandra
date: 2022-05-17
---

# RFC0006: React Native as a monorepo

## Summary

This proposal is not much as in introducing something completely new, but more to discuss a solution to an existing problem. To put simply, the current shape of the `react-native` codebase hosted in GitHub is lacking structure and consistency and this is leading to a number of issues (detailed below).

This proposal is about re-shaping the existing codebase on GitHub into what is commonly expected of a working monorepo in the JavaScript ecosystem (see, for example, [Babel.js](https://github.com/babel/babel) or [Next.js](https://github.com/vercel/next.js)).

## Motivation

To quickly explain the motivation behind this proposal, here's the current shape of the React Native codebase hosted on GitHub and corrispective npm packages:

![current status of the react-native GitHub codebase](/proposals/assets/0006-current-status.png "Current Status")

It should quickly become apparent that there are a few discrepancies:

- naming convention is not consistent (ex. `@react-native/assets`, `react-native-codegen`, `@react-native-community/eslint-plugin`)
- semver versioning is not consistent (certain packages are on a major version such as `@react-native/assets`, others are on a patch level such as `@react-native/babel-plugin-codegen`)
- semver is not aligned or coordinated with the `react-native` package semver
- no changelogs for any of these other packages

On top of this, there are some more substantial issues:

- because it's not a monorepo in which code in one folder is aware that the other package is "just next door", dependencies are fixed and pulled down from npm. For example, see how root level package.json depends on `"@react-native/polyfills": "2.0.0",` instead of the local code. ([permalink](https://github.com/facebook/react-native/blob/b331229b06dde31ff92091a163cfabb030082925/package.json#L108))
- releasing a new version of any package that is not the main `react-native` one requires an engineer manually releasing it via CLI on a local machine (which is potentially also a security issue, given how some of these packages are owned by members of the community)
- because of the root `package.json` being not just the `react-native` package but also the root/CI for the repo, during the release script for `react-native` there's some 🪄magic🪄 that happens to copy over some dependencies from `repo-config/package.json` into the root one (see [this commit](https://github.com/facebook/react-native/commit/6efb51777c620a95e457488fbc7df5c272b3c695#diff-7ae45ad102eab3b6d7e7896acd08c427a9b25b346470d7bc6507b6481575d519)). It's a weird pattern that is undocumented and I'm scared of the issues it creates, since (as all the others) `repo-config/package.json` is manually maintained.
  - for example, 0.69.0-rc0 was still depending on React 17 in the 0.69 branch; it was addressed afterwards with [this commit](https://github.com/facebook/react-native/commit/a862d9294bf22ba2304fe5d742010dc210a2ef5a)

## Detailed design

![proposal for the react-native GitHub codebase](/proposals/assets/0006-proposal.png "Proposal")

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

### Going the extra mile: Hermes

asd

### Implementation detail considerations

- This work will have to start from Meta's side (see [comments in the other proposal](https://github.com/react-native-community/discussions-and-proposals/pull/49/files#r255135557))
- Since Metro by default doesn't handle symlinks well, it could be that for RN Tester to keep working we might have to add [`@rnx-kit/metro-resolver-symlinks`](https://github.com/microsoft/rnx-kit/tree/main/packages/metro-resolver-symlinks)
- the changelog generator will need to be worked on to accomodate generating changelogs for any or all or some of the packages in the new monorepo structure

## Drawbacks

- Most of it needs to be tackled in "one fell swoop"
- Old open pull requests will likely need to be rebased and re-arranged
- Some of the native build scripts (gradle, BUCK, xcode, etc.) and CI scripts and config will need to be updated for the new code locations

- _(Adding this bullet point to easily allow comments on other drawbacks that might have not been considered)_

## Alternatives

Since this is basically a fix and not a re-invention, alternatives have not been really explored deeply. Most of the alternative takes boil down to tech choices details in ex. which tool to use to handle the monorepo setup, and not really in the final overall shape of the codebase.

## Adoption strategy

There's not really a need for an adoption strategy per se, we just ensure to complete this workstream before a given new 0.XX minor branch cut & RC process start, and communicate that as a step to upgrade to the new version that folks will have to change the name & version or certain packages to the new one ( since `react-native` will stay `react-native`, to many folks this will be transparent).

Tools like [`dep-check`](https://github.com/microsoft/rnx-kit/tree/main/packages/dep-check) or a codemod should be enough to support edge-cases in which some packages where imported directly.

No breaking changes are expected, it's just a matter of naming & semver convention re-alignment.

## How we teach this

Nothing to teach, we just need to ensure that the first version to be released after this will have the documentation and blogpost communicate accordingly the new package names (but since `react-native` will stay `react-native`, to many folks this will be transparent).

## Unresolved questions

*feel free to add something via comments*
