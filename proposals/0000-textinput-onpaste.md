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
<TextInput
	onPaste={(e) => {
		const clipboardContent = e.nativeEvent.items[0];

		if (clipboardContent.type === "text/plain") {
			console.log(`Pasting text: ${clipboardContent.data}`);
			return;
		}

		const file = {
			uri: clipboardContent.data,
			type: clipboardContent.type,
		};
		console.log("Uploading file");
		uploadFile(file);
	}}
/>
```

## Motivation

Detect `Paste` event on TextInput. This can be used to support image pasting.

## Detailed design

1:1 PR https://github.com/facebook/react-native/pull/45425

### Allow pasting images

iOS:

1. In `RCTUITextField`/`RCTUITextView` override the `canPerformAction:` method to return `YES` if the action is paste and the clipboard has images

Android: N/A (already allowed)

### Detecting paste

iOS:

1. In `RCTUITextField`/`RCTUITextView` override the `paste:` method to call the delegate adapater `[didPaste]`
2. Implement the `didPaste` method on the delegate adapater and call the delegate input `[textInputDidPaste]`
3. Implement the `textInputDidPaste` method on the delegate input to disptach the `onPaste` event

Android:

1. In `ReactEditText` override the `onTextContextMenuItem` method to call the `PasteWatcher` insatnce
2. In `ReactTextInputManager` create and set the `PasteWatcher` instance for the view (`ReactEditText`)
3. Implement the `PasteWatcher` class to to disptach the `onPaste` event

### onPaste event

Invoked when the user performs the paste action. The `items` array contains one object where `type` and `data` are the MIME type and the content of the clipboard respectively. `data` is in plaintext if the type is `text/plain`, data URI (base64-encoded) otherwise.

| Type                                                      |
| --------------------------------------------------------- |
| (`{nativeEvent: {target, items: [{type, data}]}`) => void |

## Drawbacks

-   Implementation cost: If the clipboard contains an image, the whole image is read in memory and encoded to base64

## Alternatives

None

## Adoption strategy

This is not a breaking change. Developers can adopt the feature by simply adding `onPaste` prop to their components

## How we teach this

Update the docs https://github.com/facebook/react-native-website/pull/4161

## Unresolved questions

-   Android: Pasting text using the clipboard suggestion option does not trigger the onPaste event

<video src="https://github.com/user-attachments/assets/01911237-3956-4fbe-a37d-152f83331cf5" />
