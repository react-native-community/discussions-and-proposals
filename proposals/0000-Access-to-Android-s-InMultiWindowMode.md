---
title: Access to Android's InMultiWindowMode
author:
- Sidney Ferreira
date: 2019-01-09
---

# RFC0000: Access to Android's InMultiWindowMode

## Summary

Since Nougat ( API Level 25 ) Android has both `Activity.isInMultiWindowMode` and `Activity.isInPictureInPictureMode`. Properties that so far I couldn't find ways to be aware if the app is in window mode or not.
The idea is to offer such information to the developers in order to achieve better experiences of the app.

## Basic example

- Video playing apps may use this information to better display their videos.
- Screens with specific sizes can handle better, even adding ScrollView, to achieve a better experience.

## Motivation

`Dimension.get("window")` will return the proper information of the current window, even if is in `MultiWindowMode`. But the we aren't aware of it. So we can't explore such options.
Screens that have specific sizes to achieve a better experience, the `isInMultiWindowMode` information may help the developers to tweak such sizes or even add `ScrollView` to handle it properly.
I'm currently working on an app that would have benefits of `isInPictureInPictureMode` as it has many videos playing.
When discussing about this, we noticed we had no idea how to handle this behavior.

## Detailed design

My suggestion is to just add one of the following to the `Dimension.get("window")` object:
- `type` (or similar) property with values `full`, `split` and `pip`
- Add two boolean properties `isPIP`, `isMulti`.

## Drawbacks

It is actually a really small and passive change. Can't see any drawback other than the community see it as `not interesting`.

## Alternatives

It can be achieved with an external module. But may be an overkill and the information would be split in different places.

## Adoption strategy

As it is a passive property, I see no issues with old apps.
Also if the API Level is below 25, both values will be false.

## How we teach this

All we need to do is to update the docs.

## Unresolved questions

I really want community feedback about how the information should be presented.
I personally prefer to have it as booleans but may be an "overkill". So, if we use the `type` prop, need to discuss the prop name it self and the values to use.
