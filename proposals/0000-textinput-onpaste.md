---
title: Add onPaste to TextInput
author:
    - Abdelhafidh Belalia
date: 2024-07-18
---

# RFC0000: Add onPaste to TextInput

## Summary

Add `onPaste` prop to `TextInput` component

## Basic example

```js
<TextInput onPaste={() => console.log("User pressed Paste")} />
```

## Motivation

Detect `Paste` event on TextInput. In our use case detecting paste event can be used to support image pasting.

## Detailed design

iOS:

1. In `RCTUITextField`/`RCTUITextView` override the `paste:` method to call the delegate adapater `[didPaste]`
2. Implement the `didPaste` method on the delegate adapater and call the delegate input `[textInputDidPaste]`
3. Implement the `textInputDidPaste` method on the delegate input to disptach the `onPaste` event

Android:

1. In `ReactEditText` override the `onTextContextMenuItem` method to call the `PasteWatcher` insatnce
2. In `ReactTextInputManager` create and set the `PasteWatcher` instance for the view (`ReactEditText`)
3. Implement the `PasteWatcher` class to to disptach the `onPaste` event

There is a PR for that change already https://github.com/facebook/react-native/pull/45425

## Drawbacks

None

## Alternatives

None

## Adoption strategy

This is not a breaking change. Developers can adopt the feature by simply adding `onPaste` prop to their components

## How we teach this

Update the docs https://github.com/facebook/react-native-website/pull/4161

## Unresolved questions

-   onPaste event does not expose any clipboard data. How can we add such data?
