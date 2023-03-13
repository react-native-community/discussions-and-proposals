---
title: Improve flow starting Metro.
author:
- Szymon Rybczak
date: 2022.03.13
---

# RFC0000: Improve flow starting Metro

## Summary

Right now when starting metro and the port is already in use, we'll get an error `Error: listen EADDRINUSE: address already in use :::8081`, which is not very helpful. We should improve this flow, when the port is already in use, we should ask the users if they want to start Metro on another port, or when the bundler for the project is already running, we should inform user about it.

## Basic example

<https://user-images.githubusercontent.com/63900941/224777083-91066d8a-0922-4b96-b7ac-c210f2c2d7e5.mp4>

## Motivation

This change will improve the developer experience, and will make it easier to start the bundler. All of we know what pain is to search through the terminal windows, and find the window with the running bundler.

## Detailed design

### Changes to the Core

1. Running app from Xcode / Android Studio. During the building process we're starting Metro bundler, and it would be very handy to ask user via prompt to change port if the port is already in use or inform user that bundler is already running for the project.

### Changes to the CLI

1. First part of running commands `run-ios` / `run-android` will be to check if bundler is running for the project, if it is, we should inform the user about it. Or if the port is already in use, we should ask the users if they want to start Metro on another port. It would be in the first part because we need to pass `RCT_METRO_PORT` or `-PreactNativeDevServerPort` to the build command.

## Unresolved questions

- How we should properly handle the case when running app from Xcode / Android Studio? Is it doable to ask via prompt?
- How could we better handle this on the simulator / emulator side? Right now until we don't type properly port inside DevMenu in simulator / emulator, we won't be able to connect to the bundler.
