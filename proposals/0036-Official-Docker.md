---
title: Official Android Docker for react native
author:
- Daniel Geng
date: 2018-10-5
---

# RFC0036: Official Android Docker for React Native

## Summary

Provide an official docker to the community and make the ci easier.

## Basic example

Not related.

## Motivation

Docker is a fantastic technology. With this, react native users can easily integrate ci to their code. 
Also you can easily debug react native code without setting up all the tools needed like android sdk and nodejs.
Like if you want to build RNTester after you make some change, you can just use something like this:
```bash
docker run --rm --name hei -v $PWD:/pwd -w /pwd gengjiawen/react-native /bin/sh -c "./gradlew RNTester:android:app:assembleRelease"
```

## Detailed design

I have an wip [pr](https://github.com/facebook/react-native/pull/21477), things need to do
* Fix current docker file
* Add publish docker script file
* Hook publish docker script when release a new version

## Drawbacks

Maybe a little cumbersome when update related android tools because now it use a prebuilt docker image.
But if we build the image in the circle ci, the build time will became longer.

## Alternatives
 
There are many android docker, but most of them lacks node ,ndk or buck(Although I don't use it, but some people do). 
Also react native developers needn't to find the right docker since we already provide it.

## Adoption strategy
React Native developers may need to familiar the basics of docker and ci, but this became much easier since
much more people are using it.

## How we teach this

We should add this to react native website and maybe add it to template and provide an example to demo this.

## Unresolved questions


