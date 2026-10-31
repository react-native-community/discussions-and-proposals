---
title: A supported way for libraries to read their own prop on core host components
author:
- Yevhenii Kotyrlo
date: 2026-10-31
---

# RFC0000: A supported way for libraries to read their own prop on core host components

## Summary

This RFC proposes two small, opt-in hooks. Together they let a library carry a prop that core doesn't define, such as `className`, from JSX on `View`, `Text`, `Image`, `ScrollView` and `TextInput` to C++, on iOS and Android:

1. **JavaScript:** a library registers extra attributes for the base view config before the first surface starts.
2. **C++:** a library registers a read-only observer. `ConcreteComponentDescriptor::cloneProps` calls it with the raw values of the props the library declared, plus the source props and the new props.

Neither hook changes how core parses, diffs or mounts its own props. Neither adds styling semantics to core.

## Basic example

```js
// index.js, before AppRegistry.registerComponent
import {unstable_addBaseViewConfigAttributes} from 'react-native';

unstable_addBaseViewConfigAttributes({className: true});
```

```cpp
// in the library's setup, before the first surface starts
providerRegistry.addPropsObserver(
    "View",
    {"className"},
    [](const PropsParserContext &context,
       const RawProps &rawProps,
       const Props::Shared &sourceProps,
       const Props::Shared &newProps) {
      if (const RawValue *className = rawProps.at("className")) {
        // record newProps -> className for the library's commit hook
      } else if (sourceProps) {
        // a partial update: carry sourceProps' className over to newProps
      }
    });
```

## Motivation

I'm building a CSS engine that resolves styles natively in Fabric:
- Stylesheets are compiled ahead of time.
- On the device, the engine reads `className` in C++ and finishes the tree-dependent part in a `UIManagerCommitHook` before layout.
- It writes the result into typed props.

No JavaScript runs per element. Two steps of that path have no supported API.

**Getting the prop to native.**
- `validAttributes` decides which props reach native. `createViewConfig` merges `PlatformBaseViewConfig.validAttributes` into each static view config ([ViewConfig.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/NativeComponent/ViewConfig.js#L36-L41)).
- `View`, `Image`, `ScrollView` and `TextInput` reach it through `NativeComponentRegistry.get` ([NativeComponentRegistry.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/NativeComponent/NativeComponentRegistry.js#L51-L72)). `Text` calls it directly ([TextNativeComponent.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/Text/TextNativeComponent.js#L84)).
- No API adds an attribute there. What I do today is rewrite `BaseViewConfig.ios.js` and `BaseViewConfig.android.js` in a Metro transform.
- That depends on the contents of private modules, which I re-check on every release. It is the kind of dependency on internals that the Strict API work set out to end.

**Reading it in C++.**
- **iOS:** `Props::rawProps` exists only under `RN_SERIALIZABLE_STATE` ([Props.h](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/ReactCommon/react/renderer/core/Props.h#L64-L65)), which is defined for Android alone ([react-native-flags.cmake](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/ReactCommon/cmake-utils/react-native-flags.cmake#L31-L34)). So an unknown prop is visible only inside `cloneProps` ([ConcreteComponentDescriptor.h](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/ReactCommon/react/renderer/core/ConcreteComponentDescriptor.h#L112-L181)).
- **Android:** `rawProps` holds the prop today. But once `enableExclusivePropsUpdateAndroid` is on with the Props 2.0 flags it builds on, `cloneProps` skips raw props for the components Props 2.0 covers ([ConcreteComponentDescriptor.h](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/ReactCommon/react/renderer/core/ConcreteComponentDescriptor.h#L153-L170)). That flag is expected to ship on ([ReactNativeFeatureFlags.config.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/scripts/featureflags/ReactNativeFeatureFlags.config.js#L314-L325)), and the [Binary Props RFC](https://github.com/react-native-community/discussions-and-proposals/pull/1013) builds on it.
- **Either way:** a commit hook runs after parsing, so it can't see the prop.

To see the prop, I register my own descriptor for each core component, a subclass of the core one, under the core component's name and ahead of React Native's.
- That works only because `ComponentDescriptorProviderRegistry::add` keeps the first provider registered for a handle.
- The comment beside that check says re-registering "makes no sense", and the assert against it is disabled ([ComponentDescriptorProviderRegistry.cpp](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/ReactCommon/react/renderer/componentregistry/ComponentDescriptorProviderRegistry.cpp#L16-L30)). I'd rather not build on that.

Other libraries reach the same nodes through unsupported paths:
- Unistyles 3 reaches a component's shadow node through React's internal instance handle ([ShadowRegistry](https://github.com/jpudysz/react-native-unistyles/blob/4cc87070effcfd6d130035696e3752faca032dc0/packages/unistyles/src/specs/ShadowRegistry/index.ts#L189)).
- It keeps its props in `ShadowNodeFamily::nativeProps_DEPRECATED` so that they survive React's clones ([NativeProps.h](https://github.com/jpudysz/react-native-unistyles/blob/4cc87070effcfd6d130035696e3752faca032dc0/packages/unistyles/cxx/shadowTree/NativeProps.h)).
- That field is excluded from the C++ API snapshot ([react/react-native#56350](https://github.com/react/react-native/pull/56350)). It had no lock until [react/react-native#52646](https://github.com/react/react-native/pull/52646) landed. An unlocked write to it was the cause of a production crash ([jpudysz/react-native-unistyles#1243](https://github.com/jpudysz/react-native-unistyles/issues/1243)).
- Uniwind Pro is built on the same engine ([docs](https://docs.uniwind.dev/pro/shadow-tree-updates)).

The outcome I'm after: libraries stop depending on private module contents, registration order and deprecated fields, and core keeps sole ownership of parsing.

## Detailed design

### JavaScript

`unstable_addBaseViewConfigAttributes(attributes)` is exported from `react-native`. It takes the same `AttributeConfiguration` shape as `validAttributes`.

- `NativeComponentRegistry.get` merges the registered attributes into every view config it creates, whether static or reflected at runtime. Codegen'd third-party components get them too.
- It throws if any view config has already been created, since configs are created on first use, so the result never depends on timing.
- It throws on an attribute core already defines, so a library can't change how core processes `style` or any other prop.
- Cost: the payload code walks the element's own props and looks each one up in `validAttributes` ([ReactNativeAttributePayload.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/ReactNative/ReactFabricPublicInstance/ReactNativeAttributePayload.js#L249-L253)). The first render's path does the same ([L440](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/ReactNative/ReactFabricPublicInstance/ReactNativeAttributePayload.js#L440)). An element that doesn't pass the attribute does no extra work.
- Static view config validation needs no change. It reports only native keys missing from the static config ([StaticViewConfigValidator.js](https://github.com/react/react-native/blob/f1f147405888d6ffbcab40e59a05df7aa21ddc7e/packages/react-native/Libraries/NativeComponent/StaticViewConfigValidator.js#L94-L110)).

### C++

The API goes in `react/renderer/componentregistry`, beside `add`:

```cpp
using PropsObserver = std::function<void(
    const PropsParserContext &context,
    const RawProps &rawProps,
    const Props::Shared &sourceProps,
    const Props::Shared &newProps)>;

void ComponentDescriptorProviderRegistry::addPropsObserver(
    ComponentName componentName,
    std::vector<std::string> propNames,
    PropsObserver observer) const;
```

- **Registration:** observers are registered before the first surface starts, like providers. The descriptor for that component receives them when it is created.
- **The call:** `cloneProps` calls the observers after it builds the new props, on both construction paths. A descriptor with no observer pays one empty check.
- **Declared names:** core makes the declared names readable through `RawProps::at` on both paths, including the iterator-setter path, which doesn't call `parse`.
- **Read-only:** the new props are `const`, so an observer can't change what core parsed. A library writes its results where libraries already do: in a commit hook. [RFC 1018](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/1018-cxx-strict-api.md) places commit hooks in the "for frameworks" tier, and the review of the [media query RFC](https://github.com/react-native-community/discussions-and-proposals/pull/1010) called them the place for external systems ([review comment](https://github.com/react-native-community/discussions-and-proposals/pull/1010#discussion_r3580210899)).
- **Threading:** `cloneProps` also runs for `setNativeProps` and animation updates, so an observer may be called on any thread that clones props, and has to be thread-safe.

## Drawbacks

- **Maintenance:** two new entry points to keep stable.
  - The JavaScript one is `unstable_` so it can still change.
  - The C++ one could start in the "for frameworks" tier rather than public.
- **Cost:** an observer runs on every props clone of the components it observes. Only apps that install such a library pay for it.
- **The native component boundary:** a library that writes typed props from a commit hook still works at the native component boundary. [That boundary "should be considered private, and setting styles at that boundary may not work correctly"](https://github.com/react/react-native/issues/50544#issuecomment-2790786832).
  - This RFC doesn't ask core to guarantee anything about typed props. It asks only that a library can see its own prop.
  - The risk of writing typed props stays with the library, as it does for Reanimated and Unistyles today.

## Alternatives

1. **Keep the Metro rewrite and registration order.**
   - It works today, but it rests on private module contents, and on first-registration-wins behavior whose assert is only temporarily disabled.
   - It breaks without warning on whichever release changes either.
2. **A documented way to override a core component's descriptor, like `canOverrideExistingModule` for modules.**
   - It is a smaller change, but it makes core descriptors base classes for library code.
   - The observer is composition, and it keeps core free to change its descriptors.
3. **A commit hook alone.** It can't see the prop, as shown above.
4. **A custom native component, or codegen.**
   - A `<StyledView className>` works only where the app renders it. A `View` inside any third-party library would stay unstyled.
   - Codegen describes a library's own components, not core's.
5. **Carrying the class list in `nativeID` or `testID`.** Both already mean something (native lookup, test selectors), and this would break those meanings.
6. **Keeping raw props on iOS.** That goes against the direction of the Binary Props RFC and costs memory on every node.
7. **Resolving `className` to `style` in JavaScript.** That means JavaScript work per element, and a re-render for every environment change, which is what a native engine avoids.
8. **Waiting for RFC 1018's feedback thread.** I'm glad to move this there once it opens. I'm posting now because `enableExclusivePropsUpdateAndroid` closes the Android path that exists today.

## Adoption strategy

The change is additive and opt-in, so nothing breaks. Libraries move off their workarounds when they raise their minimum React Native version. I'd write the pull requests, with Fantom and Jest tests and the docs, in whichever shape you prefer.

## How to teach this

These are library-author APIs, so the docs would be:
- a section in the native components guide;
- the API reference.

App developers never call them.

## Not part of this

- CSS, `className` or any other styling semantics in core.
- Keeping raw props on iOS.
- Any change to the commit path or to commit hooks.
- A prefix rule such as `data-*`. A library lists each attribute name it needs.

## Unresolved questions

1. Should an observer be able to return replacement props, so that a library applies element-local results while props parse, instead of cloning the node again in a commit hook? I'd start read-only.
2. Should the C++ API be in the public tier or "for frameworks"?
3. How should the declared names be made readable: as keys in the component's `RawPropsParser`, or through `RawProps::forEachItem` only when an observer exists?
4. Names, and whether the JavaScript API should also take event types.
