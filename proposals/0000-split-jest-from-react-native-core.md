---
title: Split Jest from React Native Core
author:
- Phil Pluckthun
- Cedric van Putten
date: 2025-08-17
---

# RFC0000: Split Jest from React Native Core

## Summary

This RFC proposes the removal of `react-native/jest-preset` from the core React Native package, and moving it to `@react-native/jest-preset` instead, making Jest an optional testing framework that developers can choose to integrate only if desired. This change would acknowledge the ongoing changes in test runner preferences and ease maintenance.

## Basic example

Currently, `react-native` itself contains a Jest preset and after setting up a project is available at `react-native/jest-preset`.

```json
{
  "dependencies": {
    "react-native": "0.81.x"
  },
  "devDependencies": {
    "jest": "*"
  }
}
```

The proposed change is to split this out into a separate, `@react-native/jest-preset` package, thus;

```json
{
  "dependencies": {
    "react-native": "0.81.x"
  },
  "devDependencies": {
    "@react-native/jest-preset": "0.81.x",
    "jest": "*"
  }
}
```

## Motivation

There are several key motivations for this change:

- **Maintenance Status:** While Jest 30 has now been released after a long break of major maintenance updates, the [Jest 30 includes a regression for `transformIgnorePatterns`](https://github.com/jestjs/jest/issues/15781), demonstrating that React Native isn't Jest's main focus, and that continuously maintaining a Jest preset in React Native that is part of the core package poses challenges.
- **Ecosystem Evolution:** Outside of Meta, many teams and projects have migrated to alternative testing solutions, especially for unit testing, such as Vitest. These days alternative test runners exist that are better set up to support React Native in the future, that the ecosystem signals they're starting to prefer. By bundling the Jest preset by default, we create an artificial constraint and recommendation that doesn't reflect current practices.
- **Dependency Management and size:** Including Jest by default adds significant weight to `node_modules` and projects' dependency graphs, even for projects that may be using alternative testing frameworks or no testing framework at all.
- **Testing Strategy Flexibility:** Different projects have different testing needs and requirements. Apart from unit testing, many projects implement integration or E2E tests, such as with Cypress and Maestro. This ultimately means that Jest is only fulfilling one requirement with alternatives being available.
- **Reduced Maintenance Burden:** By making Jest optional, the React Native core team can focus on the maintenance of React Native itself and signals that maintenance of the Jest preset isn't coupled to React Native releases necessarily. This is in line with the Learn Core initiative.

In short, this change affirms Jest being a choice and comes with maintenance and installation speed benefits.

## Detailed design

1. Create a separate package, `@react-native/jest-preset`
2. The current `jest` sub-folder and `jest-preset.js` move to the new package
3. `@jest/create-cache-key-function`, `babel-jest`, `jest-environment-node`, `babel-plugin-syntax-hermes-parser`
4. Update template projects, as appropriate to switch from `react-native/jest-preset` to `@react-native/jest-preset`
5. Update documentation, as needed to switch from `react-native/jest-preset` to `@react-native/jest-preset`

**Note:** There's an implicit usage of `react-test-renderer` in the `react-native/jest` folder. This package is deprecated, and moving this out of `react-native` also removes the deprecated testing recommendation from `react-native`.

### Migration Path

- Install `@react-native/jest-preset` as a dev-dependency
- Switch `preset` in your Jest configuration to `@react-native/jest-preset`

## Drawbacks

- **Migration**: Existing projects will need to add the new dependency
- **Documentation:** If any documentation assumed Jest, it will now need to point at new instructions or clarify that Jest is optional.
- **Fragmentation:** While Jest may not be the best choice for every project, this removes a default choice, which may mean beginners are faced with an additional decision.
- **Template**: Templates will have to choose whether or not to include Jest and the preset.

## Alternatives

- **Continue including Jest:** We could continue including the preset, and take on the maintenance burden associated with this, as long as development and support is revilatised.
- **Switch default testing framework:** We could replace Jest with another test framework and runner like Vitest. However, this increases maintenance burden.
- **Minimal testing interface:** We could create a framework-agnostic unit testing interface. However, `@testing-library/react-native` already exists.

This proposal was chosen partially because options for E2E testing already exist, and alternative runners can use `@testing-library/react-native`. The main complexity of replacing Jest for React Native is associated with bundling. However, as new projects make strides in supporting more projects, as long as native-platform-extension support and Babel preset transpilation of all modules is possible, a test framework can start supporting React Native. Removing the `jest-preset` from `react-native` signals to contributors to encourage other test runners to support React Native, and encourages external maintenance of new test runners.

## Adoption strategy

1. Publish `@react-native/jest-preset` in parallel to an ongoing stable release.
2. Pick a warning into `react-native/jest-preset` that informs users of the package change.
3. Drop `react-native/jest-preset` in a subsequent release.

The change to projects is an added dependency and a single line of changing the Jest preset. No codemod will be needed.

Coordination may be needed with maintainers of templates that currently set up Jest.

## How we teach this

- Documentation should reference `@react-native/jest-preset` when appropriate, but can continue referring to "the React Native Jest preset"
- Documentation should add instructions to install `@react-native/jest-preset`, where previously it may have left out instructions
- When referring to React Native testing, no assumption should be made on which testing methodology or framework developers may choose to use

## Unresolved questions

- Templates: Should templates include Jest generally, or not?
- Community: Will more test runners be encouraged to provide React Native support by this change?
- Documentation: Should any React Native documentation mention other testing frameworks and actively add new testing frameworks?
