---
title: Modal onDismiss support for Android
author:
- Bernhard Owen Josephus
date: 2024-03-04
---

# RFC0000: Modal onDismiss support for Android

## Summary

Modal `onDismiss` event is currently not supported on Android. This proposal aims to add the `onDismiss` support for Android by sending the dismiss event from the native component.

## Basic example

```js
<Modal onDismiss={() => textInputRef.current.focus()}>
```

## Motivation

The `onDismiss` event is only supported on iOS. This means that there is currently no way to know when a modal is dismissed on Android. Adding the `onDismiss` support allows us to do an action after a modal is dismissed, for example, refocusing a text input.

## Detailed design

1. Make Modal's `onDismiss` in `Modal.js` works for all platforms

This would require us to remove the iOS platform check (`Platform.OS === 'ios'`), except for `ModalEventEmitter`.

2. Send dismiss event from Android native component

First, we need to create a new event class called `DismissEvent.java`. The content will be the same as `ShowEvent.java`, except the event name will be `"topDismiss"`.

Next, we need to register a modal/dialog dismiss listener that will dispatch `DismissEvent` in `ReactModalHostManager.addEventEmitters` and `ReactModalHostView.showOrUpdate`.

Last, we need to store the `visible` props value and use it to control whether to show or dismiss the modal when the `onAfterUpdateTransaction` is called

```java
// ReactModalHostManager.java
protected void onAfterUpdateTransaction(ReactModalHostView view) {
  super.onAfterUpdateTransaction(view);
  view.showOrDismiss();
}

// ReactModalHostView.java
protected void showOrDismiss() {
  if (mVisible) {
    showOrUpdate();
  } else {
    dismiss();
  }
}
```

## Drawbacks

- Increased code size
- It could be a breaking change if an app expects the Android Modal never fires `onDismiss` event, for example:
The code uses `onDismiss` to listen for modal dismiss event on iOS and uses another (workaround) approach on Android. If the `onDismiss` is starting to fire on Android too, they would receive two dismiss events.

## Alternatives

N/A

## Adoption strategy

Check if there is an existing code that expects the Android Modal to never fire `onDismiss` event. If there is any, they need to update the code to adapt to the new changes. 

From the drawback breaking change example, they need to remove the dismiss event workaround for Android and start to fully rely on the Modal onDismiss prop.

## How we teach this

Update the docs to reflect that onDismiss is available on both platforms

## Unresolved questions

N/A
