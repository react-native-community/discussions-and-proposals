# Responding to Light/Dark Themes in React Native

## Summary
With the introduction of React Native for Windows, there needs to be a way to utilize/work with the Light and Dark themes at the JavaScript layer.

>This feature was pitched and integrated into the [React Native for Windows plugin in the PR here.](https://github.com/microsoft/react-native-windows/pull/2684)

## Motivation
Any React Native developer that wishes to follow the guidance of a Windows app will need to have access to the theme brushes, or at the very least a way to know when the system theme has changed.

There are several reasons why this is important to the React Native developer when they run their app on Windows:

- Many native Windows components will honor the Light/Dark theme changes automatically. Giving the app-developer access/knowledge of those changes reduces the "zebra" UI effect *(zebra UI = a page in an app where some controls honor a theme system, while others do not, or honor a non-complimentary one)*
- Having access to a theme changed event allows a React Native developer to better brand and customize their app for Windows, by accommodating their own styles based on the theme selected by the user *(typically set through system settings)*
- Exposing the Windows theme up to the JavaScript layer means React Native apps can blend more seamlessly into the Fluent Design system and native Windows platform more easily
- Both [Android](https://developer.android.com/preview/features/darktheme) and [iOS 13](https://developer.apple.com/documentation/appkit/supporting_dark_mode_in_your_interface) have or will have support for Dark themes, thus it is a necessity that we expose some way of detecting and handling theme changes at the RN layer.

### Scope
There are two core aspects that need to be available to the app developer in order to have a Windows Light/Dark theme sensitive app:

|# |Feature |
|:-|:-|
| 1 | Expose a way, at the JavaScript layer, to detect current theme and when the theme has changed |

## Basic examples

### Example 1 : Set up and behaviors on theme changed
In this example, we'll look at three things:

- How to set up your React Native app to be style and event sensitive to the system themes
- How to switch styles when a theme change has occurred
- Handling a theme changed event

#### Setting up your app to be sensitive to theme changes
First import the Platform API into your React Native app.

```JavaScript
import { AppThemeState } from 'react-native';
```

Create a local variable to use in style conditionals or to reference elsewhere easily.

```JavaScript
class MyAppClass extends Component {
  state = {
    appThemeState: AppThemeState.currentTheme,
  };
  ...
}
```

#### Switching styles based on the app's theme
If the app author wants to switch the style of their component manually based on the system's theme (Dark or Light), they can do so with CSS style conditionals.

```JavaScript
<TextInput
    style={[styles.inputStyle, this.state.appThemeState.currentTheme == 'dark' ? styles.darkInput : styles.lightInput]}
</TextInput>
```

#### Handling a theme changed event
In this case an app author would like to do something or preform some behavior when the app's theme has changed.

> Note: AppThemeState will be a subclass of **NativeEventEmitter**.

```JavaScript
componentDidMount() {
  AppThemeState.currentTheme.addListener('themechanged', this.onAppThemeChanged);
}

onAppThemeChanged = (event) => {
  /*Logic on theme changed goes here*/
  this.setState({appThemeState: AppThemeState.currentTheme});
}
```

## API Design Overview
A look at the APIs for the features and scope described above.

### AppThemeState.currentTheme
*The specifics of the event or API being fired/used.*

| API | Args | Returns | Discription |
|:---|:----:|:----|:--|
| currentTheme | none | string | currentTheme returns the state of the system theme that the user or native app has set. |

#### App theme enum
The ```currentTheme``` returns one of the following string values:

| Property | Type| Description |
|:---|:----:|:----|
| dark | string | A string value defining that the native app is in **Dark** theme. |
| light | string | A string value defining that the native app is in **Light** theme. |

>Could potentially need a AppThemeState.isHighContrast to detect when the system has initiated a theme for accessible users.
