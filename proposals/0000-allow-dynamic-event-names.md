---
title: Allowing Dynamic Event Names in RCTEventEmitter
author:
- Michael Heins
date: August 22, 2018
---

# RFC0000: Allowing Dynamic Event Names in RCTEventEmitter

## Summary

Currently, React Native disallows dynamically created event names when built for iOS in debugging mode. This RFC proposes a change to overcome this restriction.

## Motivation

In some applications, it is desirable to use dynamically created event names; for example, to establish an individual event channel for each wireless device based on a unique identifier for the device.  This approach works for Android, and it works for iOS built for production, but it fails for iOS built for debugging due to the way the code is implemented.

This RFC proposes a backwards compatible change to RCTEventEmitter to remove this shortcoming.

## Detailed design

The RCTEventEmitter class for iOS requires that subclasses implement a supportedEvents method, which must return a list of event names.  If the code is built with RCT_DEBUG true (but not otherwise), a check is done against this list, and if the event name is not on the list, an assert is generated, crashing the program.

It appears that the checking is a way to aid debugging by catching unexpected events. We can retain this benefit while still giving subclasses that need it a more flexible way to do the check.  What we propose is to add another method:
	- (BOOL)supportsEventName:(NSString *)name;
The default method would return NO, but a subclass which wanted to support dynamic names could override it.  
As an example, a dynamic name might for instance be a string composed of 2 parts: <module-specific string>-<device id>. The overridden supportsEventName: method could use a regular expression, or some other method, to recognize a valid event and return YES.

This change could be implemented by modifying 2 methods in RCTEventEmitter, -sendEventWithName:body: and -addListener: with an expression like this:

BOOL supportsEvent = [self supportsEventName:eventName] || [[self supportedEvents] containsObject:eventName];

In this way, the new functionality could be accommodated, but existing subclasses would continue to work the same with no changes.

## Drawbacks

None that we can think of.

## Alternatives

An alternative would be to remove the existing -supportedEvents method as a requirement, and just make it optional.  However this would mean that a subclass that needed dynamic event names would not be able to do any checking.  The proposed method allows some checking of dynamic names, and so is better than none.

## Adoption strategy

This is not a breaking change, so no adoption is required, other than to document the existence and use of the new method.

## How we teach this

We just need to provide documentation for the new method, equivalent to what is currently provided for the -supportedEvents method.

## Unresolved questions

