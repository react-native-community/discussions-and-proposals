---
title: Add possibility to onPressIn/onPressOut/onPress callbacks to trigger animation bypassing js thread.
author:
- Mikhail Boutylin
date: 03-15-2019
---

# RFC0000: Add possibility to onPressIn/onPressOut/onPress callbacks to trigger animation bypassing js thread.

## Summary

Allow to pass `AnimationConfig` and it subinstances as `onPressIn`/`onPressOut`/`onPress` properties, So the animation can be triggered directly by the UI thread.

## Basic example

```
/**
 * @format
 * @flow
 */

import React, {Component} from 'react';
import {StyleSheet, Animated, Text, View, TouchableWithoutFeedback} from 'react-native';


type Props = {};

const animation = new Animated.Value(1);

const animateIn = () => Animated.timing(
  animation,
  {
    toValue: 2,
    onComplete: () => { console.log('animateIn completed'); },
  }
);

const animateOut = Animated.timing(
  animation,
  {
    toValue: 1,
    onComplete: () => { console.log('animateOut completed'); },
  }
);


export default class App extends Component<Props> {
  render() {
    return (
      <View style={styles.container}>
        <TouchableWithoutFeedback
          onPressIn={animateIn}
          onPressOut={animateOut}
        >
          <Animated.View style={{ transform: [{ scale: animation }] }}>
            <Text>PRESS</Text>
          </Animated.View>
        </TouchableWithoutFeedback>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
});
```

## Motivation

JS thread may be overloaded, also bridge communication may have some latency. This may outcome with a not really native-like ux when building cool animations for buttons.
I think it should be better to develop a way similar to `onScroll` interpolation.
Unfortunatelly interpolation can't help here, as there no `x`/`y` changes to deal with.
So the only good sollution I see is to pass animation config directly and start it on native.
`AnimationConfig` has already `onComplete` property it could be used to trigger action on animation end.

## Detailed design

I think that this will involve designing native implementation of `AnimationConfig` invocation (if not exists yet).
Also on level of interface `onPressIn`/`onPressOut`/`onPress`. As far as I understand this api relys on `onResponderGrant`, `onResponderRelease`, `onResponderTerminate`. Those properties should be adressed as well.
I believe that native changes may happen on both ios/android.

## Drawbacks

Why should we _not_ do this? Please consider:

- ### implementation cost, both in term of code size and complexity
I think quite complex, need to be adressed on each of ios/android/js/flow/documentation sides.
- ### whether the proposed feature can be implemented in user space
I could be, but I think it will be a thirdparty native component which will make harder to integrate in the project, also I think it will rely to some internal RN api, for that reason will be quite hard to mantain with RN upgrades.
- ### the impact on teaching people React Native
Just a new possibility having `onScroll` accepting animation interpolation doesn't increase learning curve.
- ### integration of this feature with other existing and planned features
Not sure if it can affect anything.
- ### cost of migrating existing React Native applications (is it a breaking change?)
I this that api will remain backward compatible, as we only add one more interface to function signature. Need to see how flowtype will behave with that.

## Alternatives

As alternative new api like `onPressAnimation`/`onPressInAnimation`/`onPressOutAnimation` may be used

## Adoption strategy

The only issue can happen if anyone is using `$ElementType<ViewPropTypes, 'onPress'>` in there libraries / code. then interface of this method will change.

## How we teach this

I think it should be very similar to `onScroll` interpolation documentation. I don't think it's a hard to learn feature.

## Unresolved questions

In which format actually pass `AnimationConfig` via bridge.
