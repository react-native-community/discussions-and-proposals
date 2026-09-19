---
title: Customize render error handlers in React Native
author:
- Mookiies
date: Nov 21, 2025
---

# RFC0000: Make ErrorBoundary logging configurable

## Summary

`react-dom` since [React 19.0 has the ability to customize error](https://react.dev/blog/2024/12/05/react-19#error-handling)
for render errors and error boundaries via `onCaughtError`,`onUncaughtError`, and `onRecoverableError`. 
React Native does not allow for user customization of this. The current behavior is to always `console.error` and fullscreen logbox.

Relate issues:

(RN) https://github.com/facebook/react-native/issues/35483

(React) https://github.com/facebook/react/issues/15069#issuecomment-2192368527

(Expo docs calling this out) https://docs.expo.dev/router/error-handling/#work-in-progress

## Basic example

Currently, there's no way to apply these handlers and they're set [internally](https://github.com/facebook/react-native/blob/d52b9d2f8636b7c49c2f613faf48de6ccf03f6f9/packages/react-native/Libraries/ReactNative/RendererImplementation.js#L94).

The proposal is to allow passing in these handlers via `AppRegistry.registerComponent`:

```tsx
import {AppRegistry} from 'react-native';
import App from './App';
import MyErrorSDK from './MyErrorSDK';
import {name as appName} from './app.json';

AppRegistry.registerComponent(appName, () => App, undefined, {
    onCaughtError: (error, errorInfo) => {
      // Offer some way to retain default behavior if desired?
      __DEV__ && console.error(error);
      MyErrorSDK.addError(error, errorInfo);
    },
    onUncaughtError: (error) => {
        MyErrorSDK.addError(error, errorInfo);
    },
    onRecoverableError: (error) => {
        MyErrorSDK.log('Recoverable error:', error);
    },
});
```

## Motivation

Without doing this, React Native always does `console.error`, a side-effect that some users may not want. A handled error,
doesn't necessarily always warrant a console.error.

Example cases where ability to customize may be helpful:
- **Error reporting SDKS**: Many SDKs automatically pick up `console.error` calls, but manually logging errors allows 
for more context to be added to errors before reporting. Therefore, for some users it would be preferable to have manual control via these handlers. 
Or they may be living with duplicate errors.
- **Development logs**: During development, some errors are expected and handled, and do not need to be logged as aggressively. This
allows devs to control "false-positive" logs and helps avoid overly noisy and disruptive console.
- **Avoid for hacky solutions**: For users trying to workaround the above they have hacky options available like monkey patching console.error, stripping all console.error with babel, or package patching.


## Detailed design

Expose the extra configuration on via extra params `AppRegistry.registerComponent`, and ultimately pass them down into the renderer.

```ts
export function registerComponent(
  appKey: string,
  componentProvider: ComponentProvider,
  section?: boolean,
  rendererOpts?: {
    onCaughtError?: (errorValue: mixed, errorInfo: ErrorInfo) => void;
    onUncaughtError?: (errorValue: mixed, errorInfo: ErrorInfo) => void;
    onRecoverableError?: (errorValue: mixed, errorInfo: ErrorInfo) => void;
 }
): string
```

From here pass these new opts all the way through to the renderer 
[`renderApplication`](https://github.com/facebook/react-native/blob/33f783ae23dca139900767108b2ad4390ea865ca/packages/react-native/Libraries/ReactNative/AppRegistryImpl.js#L87) 
-> [`renderElement`](https://github.com/facebook/react-native/blob/33f783ae23dca139900767108b2ad4390ea865ca/packages/react-native/Libraries/ReactNative/renderApplication.js#L103) 
-> [`FabricRenderer/PaperRenderer`](https://github.com/facebook/react-native/blob/33f783ae23dca139900767108b2ad4390ea865ca/packages/react-native/Libraries/ReactNative/RendererImplementation.js#L93)
-> renderer synced from facebook/react 

## Drawbacks

- Another API for people to manage in addition to `ErrorUtils`. 
- Exposes another API surface that could cause future breaking changes should error reporting or types change. 
- Could delay app startup if people must await async SDK initialization.

## Alternatives

Unsure what alternatives exist or general plans by RN to improve error handling around error boundaries. Although it seems like they may exist 
based on some [code comments](https://github.com/facebook/react-native/blob/d52b9d2f8636b7c49c2f613faf48de6ccf03f6f9/packages/react-native/src/private/renderer/errorhandling/ErrorHandlers.js#L79-L82).

## Adoption strategy

Non-breaking change, opt-in. For Expo or react-native-web users etc. need those libraries to also adopt the new API.

## How we teach this

Communication via changelog. This concept is opt-in, and already well documented in React-DOM. We can pull from there.

## Unresolved questions

- To preserve base behavior need to expose the current behavior of `Libraries/Core/ExceptionsManager`? Maybe an extra option 
to call default behavior?
- Does this complicate migration plans to move logging into a turbo modules like mentioned in this [code comment](https://github.com/facebook/react-native/blob/d52b9d2f8636b7c49c2f613faf48de6ccf03f6f9/packages/react-native/Libraries/Core/ExceptionsManager.js#L152)?
