---
title: Use Peril on React Native Community
author:
- Orta Therox
date: 2019-1-4
---

# RFC0006: Use Peril on React Native Community

## Summary

Use [Peril](https://github.com/danger/peril) to ensure consistency across all community React Native repos. This would allow the entire community to
operate under the same rules in PRs, and can provide a way to ensure future projects have the right infra baseline.

## Basic example

Some ideas:

- Request CHANGELOG entries on repos that have [a changelog](https://github.com/react-native-community/react-native-video/blob/master/CHANGELOG.md)
- PRs from core contributors to React Native could get a label automatically set
- Peril can run a daily check that all repos:
  - Have the right license (see how we do this for READMEs [at Artsy](https://github.com/artsy/peril-settings/blob/master/tasks/closedSourceRationaleCheck.ts))
  - Have files like a Podspec

I'm happy to make a few of the rules to set it up.

## Motivation

Helping the community scale

## Detailed design

Mainly just turning it on, Peril is a GitHub app

## Drawbacks

The version that I'd add is Peril Staging, which explicitly is the version I use for testing (but Artsy, Danger and 
CocoaPods) use it. So sometimes it might be a bit unreliable when I'm working on big things. Once there is a Peril 
production, then the org can move.

## Alternatives

Danger in every repo? Or GitHub actions in every repo? It's not really the same.

## Adoption strategy

I flip a switch, make a repo with the settings ( see [an example](https://github.com/artsy/peril-settings#artsy-peril-settings) ) and add some rules. They can be code reviewed + tested like normal code.

## How we teach this

The README should contain enough links to get anyone started if they want to use and improve the setup.

## Unresolved questions

Not frmo me.
