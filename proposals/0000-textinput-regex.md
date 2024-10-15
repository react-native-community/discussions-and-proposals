---
title: Add regex to TextInput
author:
    - Abdelhafidh Belalia
date: 2024-10-15
---

# RFC0000: Add regex to TextInput

## Summary

Add `regex` prop to `TextInput` component to have user input match the specified regex.

## Basic example

```jsx
// No spaces allowed
<TextInput regex="^\S*$" />
```

```jsx
// Amount input
<TextInput regex="^\d{0,8}(\.\d{0,2})?$" />
```

## Motivation

When it comes to restricting user input, most developers opt for the re-write appraoch where user input is taken into consideration then using the `onChangeText` callback the text validated and reverted. This works for many cases but it causes a noticable flicker affecting UX. In this proposal I'm suggesting to add a new prop `regex` to validate user input on the native side and prevent change if the new input wouldn't match the specified regex.

https://github.com/user-attachments/assets/5e348ed7-f819-4e65-9189-4cbab5cad202

## Detailed design

-   iOS: Use [`textInputShouldChangeText`](https://github.com/facebook/react-native/blob/6e120b3b46c73065cd0a86082372e72e8d871946/packages/react-native/React/Fabric/Mounting/ComponentViews/TextInput/RCTTextInputComponentView.mm#L359) which is internally used by the [`shouldChangeCharactersInRange`](https://developer.apple.com/documentation/uikit/uitextfielddelegate/1619599-textfield?language=objc) method.
-   Android: Implement an [`InputFilter`](https://developer.android.com/reference/android/text/InputFilter) that uses Regex and apply it (follow same approach as [maxLength](https://github.com/facebook/react-native/blob/0902b0af75ba30ec9d4abeda71769d0685637afa/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/views/textinput/ReactTextInputManager.java#L817)).

Work in progress: https://github.com/s77rt/react-native/tree/TextInput-regex

## Drawbacks

-   Regex could be slow if the pattern is complex

## Alternatives

None

## Adoption strategy

No action is required. Developers can just use the new prop.

## How we teach this

Update the docs.

## Unresolved questions

-   Should regex flags be supported? The current implementation plan is equivalent to `/[PATTERN]/`
-   Still working on PoC for Android
