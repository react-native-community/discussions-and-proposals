---
title: Add support for Intent.SEND_ACTION in Linking
author:
- Adam Georgsson
date: 2023-10-03
---

# RFC0000: Add support for SEND intent action in Linking

## Summary

Linking have support for collecting uri from the VIEW intent action in Android. 
This proposal extends the functionality to also support SEND intent.

## Motivation

The use cases of SEND and VIEW are similar. Let's say my app is associated with opening image files. 
The Linking library will handle images opened from the file explorer in Android, but not if I open the image in another
image viewer and then press share to my app.

## Detailed design

This is an example of how it could be done in ReactInstanceManager.java:onNewIntent. 
A similar solution should be done in IntentModule.java:getInitialURL.

    /** This method will give JS the opportunity to receive intents via Linking. */
    @ThreadConfined(UI)
    public void onNewIntent(Intent intent) {
        UiThreadUtil.assertOnUiThread();
        ReactContext currentContext = getCurrentReactContext();
        if (currentContext == null) {
            FLog.w(TAG, "Instance detached from instance manager");
        } else {
            String action = intent.getAction();
            Uri uri;
            if (Intent.ACTION_VIEW.equals(action) || NfcAdapter.ACTION_NDEF_DISCOVERED.equals(action)) {
                uri = intent.getData();
            } else if (Intent.ACTION_SEND.equals(action)) {
                uri = (Uri)intent.getParcelableExtra(Intent.EXTRA_STREAM);
            }
        
            if (uri != null) {
                DeviceEventManagerModule deviceEventManagerModule =
                    currentContext.getNativeModule(DeviceEventManagerModule.class);
                if (deviceEventManagerModule != null) {
                    deviceEventManagerModule.emitNewIntentReceived(uri);
                }
            }
            currentContext.onNewIntent(mCurrentActivity, intent);
        }
    }
## Drawbacks

It may cause trouble if a project already have a solution to for handling uri from SEND actions but also using the 
Linking library for handling VIEW actions. Then they would handle each SEND action twice.

## Alternatives

I have thought of handling the SEND events in my own code and convert them to an url event that the Linking can handle.
I don't know if it would work, but it feels hacky.

## Unresolved questions

* I have not been able to test the design directly in Linking library because I did not succeed to build from source.
The concept is however already in play in another local library.

* There is also a SEND_MULTIPLE action when you share multiple files which I have not taken in consideration.

* I don't know anything about how it works in iOS. It might create a gap in functionality between the two operating systems.
