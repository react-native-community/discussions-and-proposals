---
title: Allow iOS Apps to Customize NSURLSessionConfiguration
author:
- Håkon Knutzen
date: 2019-10-21
---

## Summary

HTTP requests on iOS are handled by an instance of `RCTHTTPRequestHandler`, which internally configures its `NSURLSession`. Thus, the programmer does not have the ability to provide a custom configuration for the `NSURLSession`. As an example, the programmer may not configure the HTTP request handler with an ephemeral session configuration.

## Basic example

This proposal describes a way for providing a `RCTHTTPRequestHandler` with a custom configuration, without having to make large changes to the existing implementation, by letting the user implement a protocol whose sole purpose is providing the configuration when requested.

## Motivation

For the most part, one may not have to deal with configuring the HTTP session differently from what's already implemented in React Native (iOS). However, configuring the session differently, such as setting a custom user agent or specifying an ephemeral configuration may in some cases be desired. The motivation behind this RFC is to propose a way of enabling the programmer with an opportunity to make such changes, if deemed necessary, in a brief manner.

## Detailed design

As a class conforming to `RCTBridgeModule` will be registered for usage in React when the class is loaded by the Objective-C runtime, the modules implementing `RCTHTTPRequestHandlerConfigurationProvider` will be readily available for other classes via the React bridge.

Leveraging this mechanism, a thin protocol can be added:

```objective-c
@protocol RCTHTTPRequestHandlerConfigurationProvider <RCTBridgeModule>
-(NSURLSessionConfiguration *)sessionConfiguration;
@end
```

Given that the programmer has opted in to this feature by adding a boolean flag `ReactAppProvidesHTTPConfiguration` to the app's Info.plist file, a registered React module conforming to this protocol can be retrieved as such:

```objective-c
@implementation RCTHTTPRequestHandler
-(id <RCTHTTPRequestHandlerConfigurationProvider>)configurationProviderWithError:(NSError **)error {
  NSDictionary *infoDictionary = [[NSBundle mainBundle] infoDictionary];
  NSNumber *appProvidesConfiguration = [infoDictionary objectForKey:@"ReactAppProvidesHTTPConfiguration"];
  if (!appProvidesConfiguration.boolValue) {
    return nil;
  }
  NSArray *modules = [_bridge modulesConformingToProtocol:@protocol(RCTHTTPRequestHandlerConfigurationProvider)];
  if (modules.count > 1) {
    NSString *message = [NSString stringWithFormat:@"Only a single RCTHTTPRequestHandlerConfigurationProvider can be implemented"];
    *error = RCTErrorWithMessage(message);
    return nil;
  }
  id <RCTHTTPRequestHandlerConfigurationProvider> provider = modules.firstObject;
  if (!provider) {
    NSString *message = [NSString stringWithFormat:@"No RCTHTTPRequestHandlerConfigurationProvider module registered with the application"];
    *error = RCTErrorWithMessage(message);
    return nil;
  }
  return provider;
}
@end
```

Note that only a single `RCTHTTPRequestHandlerConfigurationProvider` is permitted. When the session is lazily instantiated in an instance of `RCTHTTPRequestHandler`, one may invoke the protocol method on the provider instance if the method above yields a non-null result. The existing session configuration may still be used if the provider instance is `nil`.

In the iOS app within which React lives, the programmer then has to implement a React module class:

```objective-c
@interface TestURLSessionConfigurationProvider : NSObject <RCTHTTPRequestHandlerConfigurationProvider>
@end
```

```objective-c
@implementation TestURLSessionConfigurationProvider

RCT_EXPORT_MODULE()

- (NSURLSessionConfiguration *)sessionConfiguration {
  // some custom implementation
  NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
  ...
  return configuration;
}

@end
```

Merely adding this class to the project will make the provider available to React.

## Drawbacks

- Only a single configuration can be used for all requests.
- The feature is not configurable via Javascript.
- Opens up for erroneously configuring the HTTP session, which may potentially result in bugs.

## Alternatives

Alternatively, one may expose a class method on `RCTHTTPRequestHandler` that can be used for assigning a configuration provider block to a static variable which can be called when the configuration should be retrieved. This approach is inspired by [AppAuth-iOS](https://github.com/openid/AppAuth-iOS/blob/66b6c77d7594ae08182cd067892875c41a6be73b/Source/OIDURLSessionProvider.m), although in the case of `AppAuth` the entire session is provided.

`RCTHTTPRequestHandler.h`:

```objective-c
typedef NSURLSessionConfiguration* (^NSURLSessionConfigurationProvider)(void);
@interface RCTHTTPRequestHandler : NSObject <RCTURLRequestHandler, RCTInvalidating>
+(void)setNSURLSessionConfigurationProvider:(NSURLSessionConfigurationProvider)provider;
@end
```

The example below illustrates how the provider block may be called if present.

```objective-c
static NSURLSessionConfigurationProvider theProvider;

@implementation RCTHTTPRequestHandler
{
  NSMapTable *_delegates;
  NSURLSession *_session;
  std::mutex _mutex;
}

@synthesize bridge = _bridge;
@synthesize methodQueue = _methodQueue;

+(void)setNSURLSessionConfigurationProvider:(NSURLSessionConfigurationProvider)provider {
  theProvider = provider;
}

...

// Convenience method for instantiating a configuration
-(NSURLSessionConfiguration *)makeConfig {
    NSURLSessionConfiguration *config;
    if (theProvider) {
      config = theProvider();
    } else {
      config = [NSURLSessionConfiguration defaultSessionConfiguration];
      // Set allowsCellularAccess to NO ONLY if key ReactNetworkForceWifiOnly exists AND its value is YES
      if (useWifiOnly) {
        config.allowsCellularAccess = ![useWifiOnly boolValue];
      }
      [config setHTTPShouldSetCookies:YES];
      [config setHTTPCookieAcceptPolicy:NSHTTPCookieAcceptPolicyAlways];
      [config setHTTPCookieStorage:[NSHTTPCookieStorage sharedHTTPCookieStorage]];
    }
    return config;
}
@end
```

This is more light weight, but it relies on setting a static variable inside of `RCTHTTPRequestHandler.m`, thus potentially being shared amongst different instances. The programmer also needs to know where in `AppDelegate` to set the configuration provider block.

## Adoption strategy

As far as I understand, this would not be considered a breaking change from the app developers perspective. However, apps that rely on previously configured session configurations would also need to take what previously was implicit into account.

It could impact further development of the internal implementation for handling network requests.

## How we teach this

Should be described as an opt-in auxillary feature.

## Unresolved questions

Configuration via Javascript could be considered preferable. However, this would require to touch on the Android implementation as well.
