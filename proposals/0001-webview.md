---
title: Extract WebView From Core
author:
- Jamon Holmgren
date: 2018-07-30
---

# RFC0000: Extract WebView From Core

## Summary

Extract the current WebView implementation from React Native Core and move it into a third party package.

## Basic example

Currently, WebView is included in React Native core:

```jsx
import { WebView } from 'react-native';
```

This would move it to an external package, like so:

```jsx
import { WebView } from 'react-native-webview';
```

(Where this third-party package will live is an open question. See below.)

## Motivation

The current WebView implementation is used by Facebook internally but not actively updated. This leads (understandably) to a lack of first-class support for this core API. By moving the component to a third party that is more invested in supporting it, there will be better support.

Additionally, with this component no longer included in core, there will be an opening for alternative third party WebView implementations, should anyone else wish to provide one.

## Detailed design

For the first release, the goal is to provide the same API, only replacing the import statement and otherwise backwards compatible.

We will also want to update generated project boilerplates, such as create-react-native-app and Ignite, to use the new package.

## Drawbacks

This is a breaking change in that if someone simply updates their React Native version in an app that uses WebView and this core API is gone, their app will break.

It also means there won't be an officially supported WebView in React Native, which may feel to some to be missing key functionality.

## Alternatives

The primary alternative is to keep the WebView in core. There are also existing third party WebViews, although none that are reliably cross-platform.

## Adoption strategy

This is a breaking change and existing React Native authors who use WebView will need to update their apps to access one of the third party solutions. For most, it should be as simple as `yarn add @infinitered/react-native-webview` and update their import statements.

## How we teach this

We will need to update the React Native documentation to point out that WebView is now moved to a third party module.

## Unresolved questions

There are a number of [WebView-related issues](https://github.com/facebook/react-native/search?q=webview&state=open&type=Issues) in the core React Native repo. We will need to have a plan to migrate those over to the new WebView repo.

Additionally, we need to decide where this new repo will live. I am willing to host it under the `infinitered` organization (this motivates me more to keep it up to date and supported), but am open to other ideas such as `react-native-community`.
