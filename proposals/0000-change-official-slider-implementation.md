# Improve the implementation of Slider

## Problem

The current 'official' implementation of slider in React Native, provided by [react-native-slider](https://github.com/callstack/react-native-slider) is full of very annoying bugs, making it unsatisfying.

Here is a list of the more dramatic ones:

- [Impossible to style the slider](https://github.com/callstack/react-native-slider/issues/254) or the [track](https://github.com/callstack/react-native-slider/issues/197), or the [thumb](https://github.com/callstack/react-native-slider/issues/169)
- Range slider is not supported (see requests [here](https://github.com/callstack/react-native-slider/issues/228) and [here](https://github.com/callstack/react-native-slider/issues?q=is%3Aissue+is%3Aopen+range+slider)
- There are plenty of bugs making it unreliable. Just take a look [at the list](https://github.com/callstack/react-native-slider/issues) and see that the issues are not just a bit annoying, but really critical

In one year, the only point that seems to have moved on on this slider is the support for Windows. The support and the follow up on all the bugs is really poor.

## Possible solution

I Propose that we make [@sharcoux/slider](https://www.npmjs.com/package/@sharcoux/slider) the new official implementation of this slider. Here is a list of why:

- It is easy to maintain as the implementation is common for all devices
- It depends only on react-native
- It also brings support to range slider and many of the propositions made in the issues of react-native-slider (like stylings).
- It is way lighter
- It is easier to understand and fix if needed as there is no native code that require people knowing those languages.

Of course I'm opened to other suggestions to fix the current state of things, but what matters to me is that we improve on this subject. Given it's ambition, it is a shame that React Native can't propose a satisfying solution for something as basic as a Slider. And I'm not saying this as a complaint. I wish React Native to grow, and become THE solution for cross-platform development, and I see this subject as discouraging for newcomers and thus, for adoption.
