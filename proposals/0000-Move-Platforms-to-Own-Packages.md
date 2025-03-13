---
title: Move iOS and Android Specific Code to Their Own Packages
author:
- Ian Meikle
date: today
---

# RFC0000: Move iOS and Android Specific Code to Their Own Packages

## Summary

Re-organize the React Native repo as a Yarn workspace monorepo, and move platform specific JS and native code for iOS and Android into their own packages.

## Basic example

Common APIs and components would remain in `react-native`:

```js
import {AppRegistry, Alert, View} from 'react-native';
```

Platform-specific components and APIs would be imported from the platform's module:

```js
// iOS
import {Alert} from 'react-native-ios';

// Android
import {Permissions} from 'react-native-android';
```

## Motivation

React Native is for more than just iOS and Android - there are renderers for Windows, macOS, and even the DOM and even more on their way.

This is analogous to when React [split into React and ReactDOM](https://reactjs.org/blog/2015/09/10/react-v0.14-rc1.html#two-packages-react-and-react-dom), and (in my opinion at least!) the separation of these modules was a benefit to the renderers of other platforms.

Splitting iOS and Android out into their own packages would make them functionally like out-of-tree platforms. The process of doing this split should hopefully strengthen support for out-of-tree platforms as a result.

The process of re-organizing these modules will, I hope, also result in more components being found that can be split out into their own repo and package as part of the slimmening.

Platform-specific APIs could drop the platform suffix from their name. `PermissionsAndroid` could just be `Permissions` for example.

React Native being a Yarn workspace might make it easier to make other modules that should be developed and released alongside the main project. It would be easier to make a more advanced RNTester app in-tree with its own set of dependencies, for example.

(**UPDATE**: I am working on a pull request to set up a monorepo, so other proposals could benefit from it regardless of this proposal's progress)

The IOS-only and Android-only files and folders at the top level of the repo could get organized into their respective packages.

## Detailed design

### Module exports:

**Components**

 * `react-native`
   * **Components**
   * `AccessibilityInfo`
   * `ActivityIndicator`
   * `ART`
   * `Button`
   * `CheckBox` (I am unsure about this one. Android-only in main tree, but generic enough other platforms could implement it. Possible slimmening candidate?)
   * `FlatList`
   * `Image`
   * `ImageBackground`
   * `ImageEditor`
   * `ImageStore`
   * `KeyboardAvoidingView`
   * `ListView` (DEPRECATED. may be removed by the time of the split)
   * `Modal`
   * `Picker`
   * `SafeAreaView`
   * `ScrollView`
   * `SectionList`
   * `Slider`
   * `Switch`
   * `RefreshControl`
   * `StatusBar`
   * `SwipeableFlatList`
   * `SwipeableListView` (DEPRECATED)
   * `Text`
   * `TextInput`
   * `Touchable`
   * `TouchableHighlight`
   * `TouchableNativeFeedback`
   * `TouchableOpacity`
   * `TouchableWithoutFeedback`
   * `View`
   * `VirtualizedList`
   * `WebView` (DEPRECATED. [Moved to its own repo](https://github.com/react-native-community/react-native-webview))
   * **APIs**
   * `Alert`
   * `Animated`
   * `AppRegistry`
   * `AppState`
   * `AsyncStorage`
   * `BackHandler`
   * `CameraRoll`
   * `Clipboard`
   * `DeviceInfo`
   * `Dimensions`
   * `Easing`
   * `findNodeHandle`
   * `I18nManager`
   * `InteractionManager`
   * `Keyboard`
   * `LayoutAnimation`
   * `Linking`
   * `NativeEventEmitter`
   * `NetInfo`
   * `PanResponder`
   * `PixelRatio`
   * `Settings`
   * `Share`
   * `StyleSheet`
   * `Systrace`
   * `TVEventHandler`
   * `UIManager`
   * `unstable_batchedUpdates`
   * `Vibration`
   * `YellowBox`
   * **Plugins**
   * `DeviceEventEmitter`
   * `NativeAppEventEmitter`
   * `NativeModules`
   * `Platform`
   * `processColor`
   * `requireNativeComponent`
   * `takeSnapshot`
   * **Deprecated PropTypes**
   * `ColorPropType`
   * `EdgeInsetsPropType`
   * `PointPropType`
   * `ViewPropTypes`

 * `react-native-ios`
   * **Components**
   * `DatePicker` (was `DatePickerIOS`)
   * `InputAccessoryView`
   * `MaskedView` (was `MaskedViewIOS`)
   * `Picker` (was `PickerIOS`)
   * `ProgressView` (was `ProgressViewIOS`)
   * `SegmentedControl` (was `SegmentedControlIOS`)
   * `SnapshotView` (was `SnapshotViewIOS`)
   * `TabBar` (was `TabBarIOS`)
   * **APIs**
   * `ActionSheet` (was `ActionSheetIOS`)
   * `Alert` (was `AlertIOS`)
   * `ImagePicker` (was `ImagePickerIOS`)
   * `PushNotification` (was `PushNotificationIOS`)
   * `StatusBar` (was `StatusBarIOS`)
   * `Vibration` (was `VibrationIOS`)

 * `react-native-android`
   * **Components**
   * `DrawerLayout` (was `DrawerLayoutAndroid`)
   * `ProgressBar` (was `ProgressBarAndroid`)
   * `Toast` (was `ToastAndroid`)
   * `Toolbar` (was `ToolbarAndroid`)
   * `ViewPager` (was `ViewPagerAndroid`)
   * **APIs**
   * `DatePicker` (was `DatePickerAndroid`)
   * `Permissions` (was `PermissionsAndroid`)
   * `TimePicker` (was `TimePickerAndroid`)


### Hypothetical project overview (non-exhaustive)

This is the *very basic* overview of what the GitHub repo could look like:

```
-| project root
--📁 {.circleci, .github, bots}/
--📁 flow/
--📁 flow-github/
--📁 packages/
---📁 react-native/
----📁 Libraries/ (All cross-platform JS)
----📁 ReactCommon/
----📁 scripts/
-----📄 launchPackager.bat
-----📄 launchPackager.command
-----📄 packager.sh
----📁 tools/ (contains bzl build-defs)
---📁 react-native-ios/
----📁 Libraries/
----📁 React/
----📁 scripts/
-----📄 ios-configure-glog.sh
-----📄 ios-install-third-party.sh
-----📄 react-native-xcode.sh
----📁 third-party-podspecs/
---📁 react-native-android/
----📁 gradle/
----📁 keystores/
----📁 Libraries/
----📁 ReactAndroid/
---📁 react-native-git-upgrade/
--📁 RNTester/
--📁 scripts/ (CI, sync, etc. scripts)
--📄 package.json (for the Yarn workspace)
--📄 {readme, contributing, CoC, other markdown docs}
```

I am not fully sure what the layout of the local-cli stuff will end up needing to be after this; with [this proposal](https://github.com/react-native-community/discussions-and-proposals/blob/master/proposals/0002-Extracted-Command-Line-Tools.md) the bulk of it will end up getting moved to another repo anyways.

ReactIOS and ReactAndroid would get entries in their `package.json` like this:

```json
{
  "rnpm": {
    "haste": {
      "providesModuleNodeModules": ["react-native-ios"],
      "platforms": ["ios"]
    }
  }
}
```

Platform-specific Flow prop types will probably need to stay bundled together in the main package. Ideally it could be set up so if you didn't have `react-native-ios` installed the iOS-only View props wouldn't show up when typechecking, for example. But I don't know if there's a way to do that with Flow yet. Would need [this issue](https://github.com/facebook/flow/issues/396) to be resolved, probably.

The native build scripts (gradle, BUCK, xcode, etc.) will all need to be updated for the new code locations. Great care must be taken to not break the existing large ecosystem of native compnents on NPM!

## Drawbacks

 - A lot of old open pull requests will need to be rebased if the codebase is re-arranged
 - The `react-native-ios` and `react-native-android` packages on NPM already have the following versions on them:
   > - `react-native-ios`: `1.0.0`, `1.0.1`, `1.1.0`
   > - `react-native-android`: `1.0.0`, `2.0.0`, `3.0.0`
   
   This will make React Native's future semver bump to `1.0.0` awkward, if not impossible. Unless it does what React did and jumps to version `60.0.0` or something like that. Or starts at version `4.0.0`, which seems kinda random. NPM also says that for various technical and security reasons, these versions can not be deleted and re-used. I've `npm deprecate`d them, which is the best I can do sadly.

   Using other names like: `@react/ios` and `react-ios` are also possibilities, since the React and React Native projects might be converging more in the future.
 - While community modules that want to cover multiple versions of RN could use the "bridge" versions of the `{ios,android}` modules to ease the transition there are packages that are not maintained that would get broken by a split like this. This is also an issue for Slimmened modules as well.
 - Even with a codemod to ease the transition, I don't imagine that transitioning codebases over will be something that is trivial. Maybe more than 2 versions should be given as a grace period before everything moves over?
 - Old documentation and tutorials will need to be updated.

## Alternatives

I believe that this is the best way forward for helping improve React Native's third party platform support, and also for helping improve the existing in-tree platforms by organizing platform-specific things into their own packages. If it's not, hopefully this will spawn discussions on better alternatives.

## Adoption strategy

There could be "bridge" versions of the `react-native-{ios,android}` modules for pre-split versions of React Native that simply import platform specific stuff from `react-native`. This would make it easier for people to make React Native addons that cover wider ranges of RN versions

For a release or so, `react-native-{ios,android}` should probably be thin modules that just re-import stuff from `react-native` before the code actually gets moved over, to help people transition.

For example, the import for `DatePickerIOS` in `react-native` would look like:

```js
  get DatePickerIOS() {
    warnMoved('DatePickerIOS', 'react-native-ios', 'DatePicker');
    return require('DatePickerIOS');
  },
```

(where `warnMoved` is a hypothetical function warning once that an api is moved to another module)

And `react-native-ios` would use Haste to import the module from `react-native`:

```js
  get DatePicker() {
    return require('DatePickerIOS');
  },
```

Alternatively RNIOS and RNAndroid could be a `peerDependency` of React Native, and it would re-import the modules from the platform modules.

Code-mods could be written to help with the transition. React and ReactDOM could be looked to as an example on how to do a transition like this as painlessly as possible.

## How we teach this

A documentation page laying out where all these components are getting moved around to should be made. This page could also document where Slimmen'd modules got moved to.

## Unresolved questions

 * Should `react-native init` auto-install `{ios,android}` by default? (I lean towards yes) If so, should there be a command line switch to not install them if the user wishes?
