---
title: Introducing E2E testing in React Native core
author:
  - Lorenzo Sciandra
  - Nicola Corti
  - Riccardo Cipolleschi
  - <Callstack folks>
date: 2023-07-06
---

# RFC0684: Introducing E2E testing in React Native core

## Summary

Testing is how we validate that the code is good, and ensure that what we build our products on top of is solid and our customers are happy.

For the longest time, in React Native core this validation has been done in a few ways:

- via Meta using the source code in their own products first
- via internal testing with custom tooling in the Meta monorepo
- via (mostly) unit testing on the CircleCI GitHub `react-native` repo
- via local testing by the release crew ahead of generating new releases
- via "pre-stable" releases (AKA Release Candidates and nightlies) that members from the community could use to test and report back

There is a great opportunity for improvement here, by introducing End-to-End (E2E) testing in the CircleCI GitHub `react-native` repository to introduce one more level of validation that aims to replace the need for (at least) the local testing from the release crew. By adding tooling that spins up the RNTester app in the codebase, and "black box" (the RNTester app is not aware that the agent using it is not a human) testing it, we can trust that the code is more solid.

This will make the codebase more stable, surface issues rapidly to the commit authors, and reduce the workload for the release crew.

## Basic example

On a basic level, the idea is to have fully automated end-to-end testing running on CircleCI on `react-native`'s every single commit to flag if any code change is inadvertently introducing a regression of some form.

The code and tests to do so will live in a new dedicated monorepo package, `packages/rn-tester-e2e` and rely on the existing `RNTester` test app.

The tooling combination selected for this is composed of [Appium](https://appium.io/) and [WebDriverIO](https://webdriver.io/) and [Jest](https://jestjs.io/).

## Motivation

The main "why" has already been explained above: introducing more solid automation to thoroughly verify that code changes don't introduce regressions.

This will lead to removing the need for the release crew to do manual local testing to validate the code before a release, which by being an human operation can lead to flukes. Additionally, this means that the time-to-release will get shorter because the release crew won't need to run those tests.

Another explanation necessary is the stack decision: the reason is that it's the combination that gets the closest to the specific requirements of React Native's unique structure. We needed to fulfil these needs:

- had to be "vanilla Jest" based
  - in order to very closely match the internal Meta monorepo tooling used by the engineers, and in doing so lower the barrier to entry to add tests for this set too
- had to not touch RNTester code (pure black box testing)
- needed to support device farm
  - while in a first phase the testing will run on CI on emulator/simulators, we are aware of the inherit flakyness so we want to start using a device farm relatively soon during the work on this effort
- easy to extend to out of tree platforms (ex. Windows, macOS)
  - RNWindows already uses [a very similar stack](https://github.com/microsoft/react-native-windows/blob/main/docs/e2e-testing.md).

With that specific combination of needs, Appium+WBIO+Jest came out as the only viable option.

## Detailed design

As already mentioned, the core stack involved is composed of [Appium](https://appium.io/) and [WebDriverIO](https://webdriver.io/) and [Jest](https://jestjs.io/).

The dedicated code configuring the tooling and the tests will be in its own private package in `packages/rn-tester-e2e`. The app used for testing is RNTester (in `packages/rn-tester`), an app already present in the codebase which contains code for every component and API with sample implementations.

In it, the `tests` folder is where the tests and referencing files all live. The substructure is as follows:

      rn-tester-e2e
      └──tests
          ├── screens
          ├── specs
          └── helpers

- `screens` -> in this folder, you will find `*.screen.js` files, where each file represents a navigation screen for RNTester. So there are 3 root ones (`apis`, `bookmarks`, `components`) and then for subscreens, there's a folder with the same name. Each of these files provide an easy way to define all elements present in said screen, so that they can be used for tests.
  - for each element of a given screen there will be a platform-specific reference in the form of

    ```js
    btnSubmitElement: Utils.platformSelect({
      ios: iOSLabel('Press to submit your application!'),
      android: androidWidget('Button', 'resource-id', 'button_default_styling'),
    }),
    ```

    we did, during early investigations, attempt to figure out more streamlined and universal solutions, such as relying on accessibility labels like `testID`s, but it created inconsistent scenarios nor every element on screen always had a `testID` available.

    *sidenote: `iOSLabel` and `androidWidget` are utility functions created while working on [the first PR](https://github.com/facebook/react-native/pull/36267). More utility functions to further simply writing tests can be added as work progresses.*

- `specs` -> this folder follows a similar 1:1 mapping to the RNTester screens, but for the tests: for each screen (or subscreen) there's a dedicated `*.test.js` file (such as `buttonComponentScreen.test.js`). Ideally, in this file the Jest tests are standard, leveraging the `*.screen.js` counterpart for the details of defining how Appium/WDIO can reach those elements on screen.
- `helpers` -> where utility code, such as methods for checking element existence, clicking interaction, etc. will live

These tests will be ran on the existing CircleCI infrastructure already set up for `react-native` on GitHub, so we will need to add two new dedicated jobs for E2E testing, `test_e2e_ios` and `test_e2e_android` and add them to the existing workflows that run on every commit.

## Drawbacks

- CI will get slower because we are introducing new jobs to the pipelines.
- during the first iteration, only on CI, tests might be flaky.
- it will take a lot of time before the whole RNTester surface area will be covered with E2E tests.

## Alternatives

There are not many tools in the end to end space when it comes to React Native; of the known alternatives, neither of the following fitted our needs:

- [Detox by Wix](https://github.com/wix/Detox)
  - greybox testing
  - no support for device farms
  - no desktop platforms support
- [Maestro](https://github.com/mobile-dev-inc/maestro)
  - yaml based testing
  - no desktop platforms support
  - no support for device farms (unclear?)

> ⚠️ This does not mean, obviously, that these are not good tools and you should not use them in your projects! But that simply for our specific requirements, they weren't viable options.

## Adoption strategy

This proposal doesn't directly affect consumers of React Native, only contributors and commit authors.

The current roadmap is as follows:

- a first PR, introducing the core infrastructure for E2E with documentation and CI integration, is prepared and merged
  - This is already in the works, check out [the dedicated PR](https://github.com/facebook/react-native/pull/36267)
- after that one is merged, an umbrella issue is created (similar to the one for [the codegen](https://github.com/facebook/react-native/issues/34872), or [Kotlin](https://github.com/facebook/react-native/issues/37708)) to ask the community to contribute by adding tests in order to cover the whole RNTester surface area.
- while the test get added and the coverage grows, there are two stretch goals that will be pursued:
  - adding a device farm as testing environment so to replace the need for using emulators and simulators on CI
  - expand the toolset to cover also for screenshot testing
    - this is current still being investigated (might end up having its own RFC), some tooling that was looked at was [`jest-image-snapshot`](https://github.com/americanexpress/jest-image-snapshot)

## How we teach this

Because this proposal doesn't directly affect consumers of React Native, the need for "teaching content" is scoped to people who wants to learn more about this infrastructure is set up and they want to contribute with more tests.

The first place where this type of documentation will be added is the readme of the dedicated new folder, `packages/rn-tester-e2e/README.md` *(you can already check out a draft of it [in the first PR](https://github.com/mateuszm22/react-native/blob/k+m/new-rn-tester-E2E/packages/rn-tester-e2e/README.md) already in the works)*. In this file, there are going to be information on how the setup works, how to test it locally and how to add new tests.

Building on top of that, an umbrella issue will be created to coordinate and provide actionable examples for how to add new tests. In it, a streamlined explanation will be provided that will then reference back to the dedicated `packages/rn-tester-e2e/README.md`.

And, of course, we can also point people to this RFC for more context/historical details.

## Unresolved questions

This proposal lives in a bit of a peculiar state because it's been written 6 months after the core investigation and decision have been taken - so there isn't much unresolved at this point. Leaving this phrase here anyway in case on the RFC someone wants to add something.
