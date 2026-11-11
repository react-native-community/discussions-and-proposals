---
title: Scroll edge effects configuration for ScrollView on iOS
author:
- SimpleCreations
date: 2026-11-11
---

# RFC0000: Scroll edge effects configuration for ScrollView on iOS

## Summary

Add an iOS-only `scrollEdgeEffects` prop to `ScrollView` (and therefore to `FlatList`, `SectionList`, `VirtualizedList` and other components built on it) that configures UIKit's [`UIScrollEdgeEffect`](https://developer.apple.com/documentation/uikit/uiscrolledgeeffect) for each edge of the underlying `UIScrollView`.

## Basic example

```tsx
// A horizontal list of chips rendered under a transparent native header.
<FlatList
  horizontal
  data={categories}
  renderItem={renderCategoryChip}
  scrollEdgeEffects={{ top: 'hidden', bottom: 'hidden' }}
/>
```

```tsx
// A content list that wants a hard edge under the navigation bar.
<ScrollView scrollEdgeEffects={{ top: 'hard' }}>{content}</ScrollView>
```

## Motivation

Starting with iOS 26, UIKit applies scroll edge effects to scroll views that underlap bars such as a navigation bar, a toolbar or a tab bar. An edge effect blurs or fades the part of the scroll view's content that is under the bar. UIKit lets apps configure the effect for each edge of every `UIScrollView` (`topEdgeEffect`, `leftEdgeEffect`, `bottomEdgeEffect`, `rightEdgeEffect`, each with `isHidden` and `style`), but React Native doesn't expose this configuration.

The automatic behavior is a good default, but apps often need to adjust it to fit their UI, for example:

- hiding the effect on scroll views that are part of the UI rather than scrolling content, such as horizontal tab strips, chips or carousels placed under a transparent header;
- choosing between the soft and hard styles to match the app's design;
- hiding the effect on a specific edge, e.g. under a custom header that already has its own background, while keeping it on the other edges.

Navigation libraries can configure the edge effects of the scroll view they consider the screen's content (e.g. `scrollEdgeEffects` in `react-native-screens`), but not of other scroll views on the screen. Any other customization currently requires custom native code, and isn't possible at all for scroll views rendered inside third-party components.

The expected outcome is a declarative way to configure the effect on any scroll view, available to both apps and libraries.

## Detailed design

### API

```ts
type ScrollEdgeEffect = 'automatic' | 'soft' | 'hard' | 'hidden';

type ScrollViewPropsIOS = {
  /**
   * Configures UIKit scroll edge effects for each edge of the scroll view.
   * Edges that aren't specified keep UIKit's default behavior.
   *
   * @platform ios
   * @supported iOS 26 or higher
   */
  scrollEdgeEffects?: {
    top?: ScrollEdgeEffect;
    left?: ScrollEdgeEffect;
    bottom?: ScrollEdgeEffect;
    right?: ScrollEdgeEffect;
  };
};
```

Values map to UIKit as follows:

| Value       | `isHidden` | `style`                              |
| ----------- | ---------- | ------------------------------------ |
| `automatic` | `false`    | `UIScrollEdgeEffectStyle.automatic`  |
| `soft`      | `false`    | `UIScrollEdgeEffectStyle.soft`       |
| `hard`      | `false`    | `UIScrollEdgeEffectStyle.hard`       |
| `hidden`    | `true`     | unchanged                            |

The shape and the values intentionally match the existing `scrollEdgeEffects` option in `react-native-screens` and React Navigation, so developers who already configure edge effects at the screen level don't need to learn a new vocabulary.

### Native implementation (iOS)

- Add the prop to `ScrollViewNativeComponent`, the TypeScript and Flow types, and `BaseScrollViewProps` in C++ (as an iOS-only prop, similar to other iOS-specific scroll view props).
- In `RCTScrollViewComponentView`, apply the configuration in `updateProps` when the value for an edge changes, guarded by `@available(iOS 26.0, *)`. On older iOS versions, the prop is a no-op.
- React Native only writes to an edge's `UIScrollEdgeEffect` when that edge has been configured through the prop. Scroll views that don't use the prop are left untouched, so libraries that configure edge effects natively keep working as they do today.
- When an edge that was configured is reset to unspecified (or the view is recycled), React Native restores UIKit's defaults (`isHidden = false`, `style = automatic`) for that edge, so configuration doesn't leak between recycled views.
- Components built on `ScrollView` get the prop through the existing props pass-through, without extra work.

### Android and other platforms

The prop is ignored. Android doesn't have an equivalent concept (Android's `EdgeEffect` is the overscroll effect, which is unrelated and already covered by `overScrollMode`).

## Drawbacks

- It adds iOS-specific API surface to `ScrollView`, only meaningful on iOS 26 and later.
- Apps and libraries can now configure the same scroll view from both JavaScript and native code (e.g. a navigation library configuring the content scroll view natively). The design limits this by only writing configured edges, but if both configure the same edge, the last writer wins.
- The name can be confused with Android's `EdgeEffect`, which is the overscroll effect. This needs to be addressed in the documentation.
- Apple may add more styles or options to `UIScrollEdgeEffect` in the future, which would require extending the union type.

## Alternatives

- **A single boolean prop (e.g. `scrollEdgeEffectsHidden`).** This covers the most common need (opting out) with a minimal API, but doesn't allow choosing a style or configuring edges independently, e.g. hiding the top effect while keeping the bottom one under a tab bar. It could be a first step if the per-edge configuration is considered too broad.
- **Separate props per edge (e.g. `topScrollEdgeEffect`).** Equivalent in capability, but adds four props instead of one, and diverges from the shape already used by `react-native-screens` and React Navigation.
- **Handling more cases automatically in React Native.** React Native can't know whether a scroll view is the screen's content or part of the UI, or which style fits the app's design, so this decision belongs to the app.
- **Leaving it to navigation libraries.** Libraries can only reliably configure scroll views they can find, and scroll views rendered by other libraries are out of their reach. A prop on the scroll view itself works regardless of who renders it.

A similar change was proposed in [react/react-native#55037](https://github.com/react/react-native/pull/55037), which was closed by its author without review. This RFC aims to agree on the API shape before an implementation is submitted again.

## Adoption strategy

The change is additive and non-breaking: scroll views that don't use the prop behave exactly as they do today. Apps can adopt it on individual scroll views where the automatic behavior is wrong. Libraries that render internal scroll views (e.g. tab bars or carousels) can set it on their own scroll views, or forward it from their props.

## How we teach this

- Document the prop in the iOS section of the `ScrollView` props, with a short explanation of what scroll edge effects are, when UIKit applies them, and a link to Apple's documentation.
- Mention that the prop is also available on `FlatList`, `SectionList` and other components built on `ScrollView`.
- Clarify that the prop is unrelated to Android's overscroll effect.

## Unresolved questions

- Is the per-edge configuration preferred over a simpler boolean opt-out?
- Should `left` and `right` be named `start` and `end` to respect RTL layouts, or should they stay physical to match UIKit?
- React Native already hides edge effects on inverted lists internally. Should an explicitly configured edge take precedence over that?
