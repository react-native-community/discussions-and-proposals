---
title: Use Kotlin DSL in gradle scripts
author:
- Dulmandakh Sukhbaatar
date: 2018-08-11
---

# RFC0003: Use Kotlin DSL in gradle scripts

## Summary

Convert gradle scripts in ReactAndroid, and maybe RNTester, to Kotlin DSL.

## Motivation

Due to dynamic nature of Groovy language and lack of proper IDE support, its hard to make changes to gradle scripts because you have to run it to make sure that it works correctly. But running gradle scripts takes some time, and requires a developer to setup SDK and NDK, even for a trivial change.

In 2016, Gradle introduced Kotlin programming language support in gradle scripts and it'll become stable in Gradle 4.10. Here is an excerpt from [announcement blog post](https://blog.gradle.org/kotlin-meets-gradle)

> Because Kotlin is a statically-typed language with deep support in both IDEA and Eclipse, it could give Gradle users proper IDE support from auto-completion to refactoring and everything in-between. And because Kotlin is rich with features like first-class functions and extension methods, it could retain and improve on the best parts of writing Gradle build scripts—including a clean, declarative syntax and the ability to craft DSLs with ease.

Thanks to static types and IDE support of Kotlin, IDE will offer following benefits, so that you don't have to run the gradle scripts.

* auto-completion and content assist
* quick documentation
* navigation to source
* refactoring and more

Also, nowadays there are more Android developers who knows Kotlin than Groovy.

## Detailed design

Convert only gradle scripts in ReactAndroid, and maybe RNTester, to Kotlin DSL, so we don't introduce breaking changes to RN based projects.

* Migration documentation https://guides.gradle.org/migrating-build-logic-from-groovy-to-kotlin/
* Samples https://github.com/gradle/kotlin-dsl/tree/master/samples

## Drawbacks

Currently, RN use Gradle 4.4 which has partial but enough support for Kotlin DSL. Until we have 4.10, which has stable Kotlin DSL, we have to make sure that gradle scripts works and might require changes to work with landed Gradle version.

## Alternatives

Stay with Groovy based gradle scripts.

## Adoption strategy

This is not a breaking change because it affects only gradle scripts used to build RN for Android.

I converted build.gradle and settings.gradle in root directory, see at
https://github.com/dulmandakh/react-native/tree/kotlin-dsl

## How we teach this

This change won't affect RN based projects and RN developers, but RN core contributors. 