---
title: Autolinking local modules
author:
- Szymon Rybczak
- Satyajit Sahoo
date: 21-08-2023
---

# RFC0000: Autolinking local modules

## Summary

This RFC proposes a new feature to [React Native CLI](https://github.com/react-native-community/cli/) where we can create a folder containing local native modules and views and have them linked automatically. Proposed solution will simplify the space of creating local native modules and views and using them later in the app.

## Motivation

Currently, when we have local native modules, we have to create them under `android` and `ios` folders and write the code for registering those packages manually. This poses a few problems:

1. The new modules become part of the `android` and `ios` source code—which means upgrading the project to a newer React Native version is harder as it also needs to consider those changes,
2. The module boilerplate is written manually, that's the place where users can make mistakes and it takes a lot of time to write one,
3. The code for registering the modules is additional boilerplate which is not required for third party packages (because they're automatically linked),
4. The workflow for creating new architecture modules without creating packages is not streamlined,
5. It's challenging to open-source the local module, as it requires creating a dedicated library setup that adds extra friction to the process.

## Detailed design

To automate the process of creating native modules or views we need to implement a few things.
First we need to introduce a new feature to the `create-react-native-library` package to create local modules. `create-react-native-library` will detect if the command has been run under React Native project if yes, it will create a local module under `local_modules` directory. CRNL will contain templates for creating native modules and views, also templates for turbo modules and fabric views.

Second thing is to expand the abilities of autolinking to also link local modules. React Native CLI will link modules placed under `local_modules` directory and CLI will re-use logic that we have right now for adding native dependencies. See [here](https://github.com/react-native-community/cli/blob/main/docs/dependencies.md) for more references.

Last thing is to adjust codegen script in React Native Core to also look under `local_modules` directory, because right now it looks under `node_modules` and dependencies specified in `react-native.config.js`.

With these few things implemented - process of creating local modules will look:

1. Run `npx create-react-native-library` under React Native app, to create module,
2. Rebuild app. (install pods in case of iOS),
3. Import module:

    ```jsx
    import React from 'react';
    import { View } from 'react-native';
    import helloWorld from './local_modules/hello_world'; // import module

    helloWorld(); // call module

    const App = () => {
      return <View />
    };

    export default App;
    ```

Project structure will look:

```
- App.tsx
- package.json
- local_modules
- hello_world
  - hello-world.podspec
  - index.ts (uses React Native APIs to exports module / view)
  - ios
    # native module code
  - android
    # native module code
```

Putting the files under a single directory under CRNL governance, makes it straight-forward to convert to a reusable library, and increases the chances for the module authors to open source their efforts. Separating all the local modules via different folders in one place will also make it easier to organize them and track the changes.

## Alternatives

- [react-native-coco-loco](https://github.com/jamonholmgren/react-native-colo-loco), it's a great solution - but it integrates with native code, proposed solution will autolink local modules and won't touch native code at all - which makes upgrades simpler and faster.
- [Expo Local Modules](https://docs.expo.dev/modules/get-started/#creating-the-local-expo-module) Expo has a similar autolinking directive for modules that are using Expo Modules API.

## Drawbacks

There'll be more files inside modules placed in `local_modules`, such as `react-native.podspec` etc. But those are files that are not likely to be edited.

## How we teach this

- Adding relevant information to React Native docs,
- Blog posts.
