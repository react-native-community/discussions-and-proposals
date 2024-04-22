---
title: Adjusts Font Size To Fit for TextField
author:
- Lukas Lindqvist
date: 2023-08-15
---

# RFC0699: Adjusts Font Size To Fit for TextField

## Summary

Support both EditField(android) and UITextField(ios) built-in approach to adjustsFontSizeToFit similar to the existing approach for Text-component

## Motivation

When using TextInputs it is sometimes good to be able to have an input which starts big and then shrinks when inputing. This is used for example in iOS Calculator app. 

## Detailed design

EditField supports this out of the box via the inherited constants `AUTO_SIZE_TEXT_TYPE` and so does UITextField via the `adjustsFontSizeToFitWidth` boolean and `minimumFontSize` variable