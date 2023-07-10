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

* via Meta using the source code in their own products first
* via internal testing with custom tooling in the Meta monorepo
* via (mostly) unit testing on the CircleCI GitHub react-native repo
* via local testing by the release crew ahead of generating new releases
* via "pre-stable" releases (AKA Release Candidates and nightlies) that members from the community could use to test and report back

There is a great opportunity for improvement here, by introducing End-to-End (E2E) testing in the CircleCI GitHub react-native repository to introduce one more level of validation that aims to replace the need for (at least) the local testing from the release crew. By adding tooling that spins up the RNTester app in the codebase, and "black box" (the RNTester app is not aware that the agent using it is not a human) testing it, we can trust that the code is more solid.

This will make the codebase more stable, surface issues rapidly to the commit authors, and reduce the workload for the release crew.

## Basic example

If the proposal involves a new or changed API, include a basic code example. Omit this section if it's not applicable.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Please focus on explaining the motivation so that if this RFC is not accepted, the motivation could be used to develop alternative solutions. In other words, enumerate the constraints you are trying to solve without coupling them too closely to the solution you have in mind.

## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with React Native to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

## Drawbacks

Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React Native
- integration of this feature with other existing and planned features
- cost of migrating existing React Native applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? Why did you select your approach?

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

## How we teach this

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?

How should this feature be taught to existing React Native developers?

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still TBD?
