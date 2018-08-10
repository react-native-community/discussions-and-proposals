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

Facebook does not use CLI provided with React Native, which makes it hard to maintain the project. There is no clear owner and issues / pull requests get stack in the sea of other, non-CLI related reports.

It is also hard to keep moving on the CLI as every additional dependency or code change is subject to the heavy review process and sync.

Having a separate package with me (and Callstack) being strategic maintainers would allow us to iterate on the design of CLI even more and make sure we have the best tooling out there.

Now, if you have been following React Native development for a few years, you know I have been always super excited about CLI in general. I did many talks in that field and lead efforts of creating RNPM and then, merging it to the core as `react-native link` together with Alexey Kureev. Link, and RNPM, in general, was just the first step on our way to meet that high expectation of high-quality tools. Unfortunately, due to the aforementioned constraints and lack of time we were never able to complete our long-term strategy.

I would like to take the slimmering process as an opportunity to revisit that effort and start working on it once again, together with the community.

## Detailed design

For the first release, the goal is to just extract out the CLI into a separate package. Since every command is a standalone module, we would like to use Yarn workspaces to build a modular architecture around it.

Not many people know, but not only CLI allows you to provide your custom commands but also to override the built-in ones. That makes it even more obvious to create a package-oriented setup. The similar setup is already followed by React and Metro and we would take learnings from there.

Apart from reorganizing the architecture, we would like to create a website that describes the features present in the CLI - many of them, making it much easier to work at a scale. Things like support for different locations of iOS/Android projects, linking assets from 3rd party modules are great features that only selected people were fortunate to learn about.

As a next step, we would like to focus on reinvestigating every single piece of CLI to see what we can do better these days. That includes looking once again into the design of the scaffolding system (that is great, but has its limitations), lock-in to Metro (because why it should be the forced default?) and rewriting link to support modern languages and new module system. 

The list of features is much longer and as a someone who believes tools are to give framework success on the market, I want to make sure we get the best tooling possible.

## Drawbacks

There are no drawbacks, the whole migration process happens as a part of one of the future React Native versions w/o users noticing. The separate Github repository is just for better tracking of issues.

In the future, we might consider decoupling these two, but at the time of writing this document, I do not have answers to many questions arising from such a move.

The CLI would still remain official, to go, solution that would end up living under `react-native-community` with a note, that Callstack team is working on it as a part of their OSS programme and we invite everyone to join our planning meetings.

## Alternatives

This is the standard and default tool for developing React Native apps that many other tools, like CRNA and Ignite depend on. We do not plan on competing with anyone, but providing a good set of built-ins
that makes it easy for others to innovate on top.

## Adoption strategy

Rolls out automatically as a part of one of the future RN releases, when ready.

## How we teach this

We will need to point to new CLI documentation which describes all additional features. Rather than that, it's not needed.
