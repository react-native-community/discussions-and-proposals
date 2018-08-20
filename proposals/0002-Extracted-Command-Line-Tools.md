---
title: Extract CLI from Core
author:
- Mike Grabowski
date: 2018-08-10
---

# RFC0000: Extract CLI from Core

## Summary

Extract the current CLI implementation from React Native Core and move it into a third party package.

## Basic example

Command line tools in React Native are split into two, separate pieces. 

First one is the `react-native-cli` package that we prompt users to install globally, for the first time they actually interact with the framework. This package, by the design, is not meant to be frequently updated, which eliminates many annoying issues that global packages are infamous for.

The second one is `local-cli` package which is the actual source code of the CLI. It is a set of multiple commands, each of them grouped into a standalone module, that are loaded altogether when the CLI is spawned. 

The biggest part of the `local-cli` is its `core`, which is capable of analyzing your source code. It provides a context object for all the commands, that contains necessary information about the location of your source files, available platforms and more. Commands such as `link`, `unlink` or `run-ios` use that information to perform their work.

This proposal suggests no changes to the end user in its first iteration. We would simply couple new package with React Native version, just like we do with Metro. That makes it possible to still make React Native version specific changes to the CLI itself. 

## Motivation

Facebook does not use the CLI provided with React Native, which makes it hard to maintain the project. There is no clear owner and issues / pull requests get stuck in the sea of other, non-CLI related reports.

It is also hard to keep moving on the CLI as every additional dependency or code change is subject to the heavy review process and sync.

Extracting it out would be the first step to make React Native CLI agnostic, allowing other tools (including Webpack via Haul) to become first-class citizens and use common design infrastructure.

## Detailed design

The goal of this proposal is to extract the CLI out of the React Native repository into a separate project, called `react-native-cli`, that would become a direct dependency of React Native. It is similar to the extraction process of `Metro`.

Since `metro` isn't required anywhere else outside of `local-cli` folder (where the business logic of the CLI is located), it would be removed from the React Native dependencies and become the depdendency of newly created `react-native-cli`.

## Drawbacks

Every time the new release of React Native is cut, few dependencies must be updated (`metro` in particular). Due to the way Facebook develops their products internally using monorepo, the version of `metro` must be set manually by the Metro team right after they release their version to the `npm` registry.

After extracting out `react-native-cli` to a separate repository and `metro` becoming its dependency, Metro team would have to update the version inside that project instead. React Native CLI maintainers would then make sure to update the versions when cutting new release.

## Alternatives

There's Haul - an open source project by Callstack - in development that uses Webpack instead of Metro to bundle React Native code. It is a different project having different goals. Unfortunately, due to tight-coupling of `local-cli` with React Native, it became hard to integrate these two together.

## Adoption strategy

The CLI would roll out as an independent package automatically with one of the upcoming React Native versions.

## How we teach this

Clear communication in the changelog. Printed instructions on how to fill in bug report right after the CLI ends up with non-zero exit code.
