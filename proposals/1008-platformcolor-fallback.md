---
title: Raw color fallback for PlatformColor
author:
- Peter Abbondanzo
date: 2026-07-08
---

# RFC1008: Raw color fallback for PlatformColor

## Summary

`PlatformColor('someToken')` hands a name to native and asks it to resolve a semantic, system, or theme color at runtime. When that name isn't in the installed binary, what comes back is a mess: sometimes a crash, sometimes a transparent view, and which one you get depends on the platform, the architecture, and even which style prop you happened to use.

This adds an optional raw color fallback:

```js
backgroundColor: PlatformColor('?myThemeColor', {fallback:'rgba(255, 255, 0, 1)'})
```

The whole trick is that the fallback is **lazy**. It crosses the bridge as a plain unparsed string and just sits on the color object. Native doesn't look at it, and doesn't parse it, until every name has failed to resolve. If the token resolves, the fallback never gets touched, so no costs are paid.

## Basic example

```js
import {PlatformColor, StyleSheet} from 'react-native';

const styles = StyleSheet.create({
  card: {
    // The iOS semantic color, or a plain near-white if the OS/app is too old.
    backgroundColor: PlatformColor('secondarySystemBackground', {fallback:'#f2f2f7'}),
  },
  progress: {
    // The Android theme attribute, or a concrete brand color if the token
    // isn't in the installed APK yet.
    color: PlatformColor('?myBrandProgress', {fallback:'#000000'}),
  },
});
```

If the token in the first argument position resolves, you get the token. If it fails tor esolves, you get your fallback color, not a crash and not an invisible view.

## Motivation

### What breaks today

More and more RN apps ship their JavaScript through dynamic update pipelines (CodePush-style), and that JS routinely lands ahead of the native binary. So, a fresh bundle can name a color token that only exists in a newer build than the one sitting on someone's phone, and `PlatformColor('?newToken')` hits a resolver that's never heard of it.

This isn't hypothetical--we've run into this exact issue in production. One bundle referenced a brand new design-system token that wasn't in the native binary yet, and every device that picked up the JS crashed. A temporary fix was to ship a sidecar bundle with a temporary literal for the new token, but that's a stopgap and it prevents apps from quickly rolling out new tokens.

Here's what happens on each platform when you reference a color that does not exist:

- **iOS, old arch (Paper).** `+[RCTConvert UIColor:]` returns `nil` and logs a convert error. "No color," which usually renders as transparent.
- **iOS, new arch (Fabric).** `RCTPlatformColorFromSemanticItems` hands back `UIColor.clearColor`. Transparent, silently, on purpose.
- **Android, old arch (Paper).** `ColorPropConverter` returns `null` per path, and when every path misses it **throws `JSApplicationCausedNativeException`** for generic `@ReactProp` color props, `boxShadow`, and gradients. That's the crash. `backgroundColor` and `shadowColor` are the confusing exception: they go through a 3-arg `getColor(..., defaultInt)` that swallows the error and returns `0`, so those paint transparent instead, and `android_ripple` falls back to `colorControlHighlight`.
- **Android, new arch (Fabric).** `FabricUIManager.getColor(...)` returns `0`, no throw. Transparent again.

iOS does already have a fallback, but for a different case. A name RN knows about whose selector isn't available on the running OS falls back to a hard-coded ARGB (`RCTFallbackARGB` / `fallback-argb`). That only covers a fixed set of standard UIKit colors though. It does nothing for app or design-system tokens.

### Why the stuff we already have doesn't cut it

We've got a few things that look adjacent, and none of them actually solve this. Multi-name `PlatformColor(a, b)` is an ordered fallback, but only across native names. Every entry still has to resolve natively, and if none of your tokens exist in the old binary you're right back where you started.

`DynamicColorIOS` is a light/dark/high-contrast switch, not a fallback chain, it wants both `light` and `dark`, and it's iOS-only. There are build-time stopgaps (some design-system tooling ships a temporary literal for a brand-new token through a self-expiring sidecar, and there's a crash guard that turns the Android throw into a silent miss), but those are per-org, per-surface band-aids that never add up to a real cross-platform primitive a product author can just reach for.

## Detailed design

### The API

`PlatformColor` picks up an optional trailing raw color:

```js
PlatformColor(...names: string[], {fallback?: RawColor}): ColorValue
```

Leading args are native names, exactly like today, tried in order, first hit wins. The trailing arg, if an object type containing `fallback` key, is the fallback and only comes into play if the previous tokens have not resolved. The fallback cannot exist in any other argument position.

```js
PlatformColor('?token')                            // unchanged: name only
PlatformColor('?a', '?b')                          // unchanged: two names, ordered
PlatformColor('?token', {fallback:'#f2f2f7'})    // new: name + raw fallback
PlatformColor('?a', '?b', {fallback:'#000000'})      // new: two names + raw fallback
```

By using an object type, this avoids any ambiguity about whether the last arg is a name or a fallback. You could imagine a bare color like `'red'` as a name--should that attempt to resolve to the native color `red`? Or should it fall back to the raw color `#ff0000`? The object form clears that up, and it also makes it easier to expand the PlatformColor API in the future to support alpha, prominence, etc. as implemented by https://github.com/react/react-native/pull/55012.

### The lazy bit (this is the core of the proposal)

The fallback lives in a **new key holding a raw string** on the opaque color object, and JS never parses it:

- iOS: `{semantic: string[], fallback?: string}`
- Android: `{resource_paths: string[], fallback?: string}`

Compare that to `DynamicColorIOS`, whose legs get run through `processColor` at construction. We're deliberately not doing that. The fallback string crosses the bridge byte-for-byte as the author wrote it, native holds onto it, and it only gets parsed in the single branch where every name failed.

Why lazy and not pre-processed in JS? Three reasons, and the first is the one that matters. The happy path is the token resolving, and in that case native never even reads the fallback, so there's zero wasted work and zero bytes spent turning a literal into a number nobody uses. Second, it doesn't couple you to whatever version of the JS color parser shipped this release: the literal you wrote is the literal that ships, and `normalize-colors` can't quietly re-encode it out from under you. Third, it's just easier to debug. `{fallback: '#000000'}` on the wire tells you exactly what's going on, where a pre-resolved `0xff000000` doesn't.

### JS layer

Files: `Libraries/StyleSheet/PlatformColorValueTypes.ios.js`, `PlatformColorValueTypes.android.js`, `PlatformColorValueTypes.macos.js`, `PlatformColorValueTypesIOS.ios.js`, plus `StyleSheetTypes.js` for types.

`PlatformColor` splits its args into names and an optional fallback and drops the raw fallback string straight onto the object:

```js
// android.js (illustrative)
export const PlatformColor = (...args) => {
  const fallback = args.length > 1 && typeof args[args.length - 1] !== 'string' ? args.pop().fallback : null;
  const names = args;
  return {
    resource_paths: names,
    ...(fallback != null ? {fallback} : {}),
  };
};
```

### iOS

Two resolvers, and both need the same "on miss, parse the fallback" branch.

- (Optional) Old arch, `React/Base/RCTConvert.mm`: In `+[RCTConvert UIColor:]` the `semantic` loop tries each name. Where it currently falls through to `RCTLogConvertError` / `return nil`, read the dictionary's `fallback` string first and parse it to a `UIColor`. Only log and return nil when there's no fallback, so existing callers don't move. Marking as optional because the old arch is going away.
- New arch, `ReactCommon/react/renderer/graphics/platform/ios/...` (`PlatformColorParser.mm`, `RCTPlatformColorUtils.mm`, `HostPlatformColor.mm`): `parsePlatformColor` type-guards `semantic` as `vector<string>` today, and `RCTPlatformColorFromSemanticItems` returns `UIColor.clearColor` on a total miss. Read `fallback` from its own map key and parse it in that miss branch instead of returning clear. Since `fallback` is a separate key, the `semantic` type guard doesn't change.

### Android

Android actually has a nice choke point. `ColorPropConverter.resolveResourcePath` (`ReactAndroid/.../com/facebook/react/bridge/ColorPropConverter.kt`) is shared across both architectures, with the ordered loops living in `getColorInteger`/`getColorInstance` on old arch and `FabricUIManager.getColor` on Fabric.

- Old arch: In `getColorInteger` and `getColorInstance`, when the `resource_paths` loop runs out, read `fallback`, parse it, and return that **instead of throwing `JSApplicationCausedNativeException`**. Keep the throw for the no-fallback case so nothing existing changes.
- New arch: The C++ `parsePlatformColor` (`ReactCommon/.../graphics/platform/android/.../PlatformColorParser.h`) only reads `map["resource_paths"]` and defaults to `0` right now. It needs to read `fallback` and thread it through so `FabricUIManager.getColor` returns the parsed fallback rather than `0` on a total miss. `NativeDrawable.h` and `HostPlatformViewProps.cpp` (ripple drawables) rebuild the same object and need the key too.

### Backward compatibility

Opt-in and additive all the way down. `PlatformColor(...names)` with no fallback builds the exact same object it does today and behaves identically, so nobody's affected until they add one. Old binaries just ignore the new key: `fallback` is an optional field on an object that already crosses as a dictionary, so a new JS bundle carrying a fallback running on an old binary quietly degrades to today's behavior. That's fine, it's no worse than the status quo, and the fallback lights up the moment the binary understands it.

## Drawbacks

The big one is that **native has to parse colors now**, which is the direct cost of going lazy. That logic moves out of one shared JS implementation and into native, ideally one shared C++ parser for the two Fabric paths plus thin legacy wrappers, but it's still more code in more places than a JS pre-process would be. **Format support is narrower** than JS colors until that parser grows functional notation.

## Alternatives

1. **Pre-process the fallback in JS** (run it through `processColor` at construction, ship a resolved number). This was in my first draft. You get full JS color-format support for free and need no native parser, which is genuinely tempting. We're not doing it because it does tons of wasted parsing work on the happy path when the fallback is never used. Lazy is the deliberate call.
1. **Supporting inline fallback colors** (`PlatformColor('?x', fallback)`). This ended up being ambiguous, and for reasons that I highlighted above, it may regress in unexpected ways for those who use platform colors that share with parsable names.
1. **A separate function** (`PlatformColorWithFallback(name, fallback)`). Leaves `PlatformColor`'s signature alone but splits the API and doesn't compose with multi-name usage as cleanly. Not a huge fan but it's a reasonable alternative.
1. **Build-time sidecar only** (the existing design-system tooling approach). Proven, but quite a pain to maintain and will always force a complicated system that forces new color tokens to lag behind.
1. **Global "never crash, always transparent on miss."** Makes the miss consistent but hands the author zero control and hides the problem. It's how things mostly work today and is fine as a separate safety net, not a substitute for a real fallback.
1. **Do nothing, lean on dynamic-update gating.** Necessary defense in depth, but it does nothing for the many legitimate cases where JS correctly leads native, and those authors still need a runtime-safe color.

## Adoption strategy

Nobody has to migrate, existing code keeps working. It pays off in three places.

1. **Design systems and token codegen** (anything that emits `PlatformColor`) should emit `PlatformColor('?token', {fallback:'#literal'})` with the literal set to the token's resolved default. That turns the self-expiring sidecar into a permanent, runtime, cross-platform mechanism, and it makes the crash-guard workaround unnecessary for any token carrying a fallback.
1. **Apps that ship JS dynamically** add fallbacks to any `PlatformColor` that can go out in a bundle ahead of native. App authors can drop a conservative fallback onto existing call sites (the current resolved value, or `transparent` as an explicit opt-in).

## How we teach this

- Update the `PlatformColor` docs page: the fallback arg, the v1 formats, and a one-liner rule of thumb ("put a raw fallback on any token you ship over the air").
- Add an ESLint rule (and/or codemod) that nudges toward fallbacks on `PlatformColor` calls referencing app or design-system tokens, and flags a bare CSS keyword sitting in a name position.
- Cross-link from the `DynamicColorIOS` docs with the per-leg composition example so people see the two work together.

## Unresolved questions

1. Should the no-fallback miss get unified to "transparent plus a warning" across all platforms and arches as a companion change, so behavior is predictable even without an explicit fallback? Separable from this.
1. Should a firing fallback be observable by default (a dev warning, a sampled production signal) so a real missing-token bug doesn't stay hidden?
1. Do we ever allow a nested `PlatformColor` (or `DynamicColorIOS`) as the fallback, or keep it strictly raw? v1 is strictly raw. The name array already covers "try another token," so nesting buys nothing.
