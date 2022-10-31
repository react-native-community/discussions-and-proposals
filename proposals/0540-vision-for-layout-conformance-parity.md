
# Vision for Layout Conformance/Parity

React Native models its layout and styling support after web browsers, offering a subset of CSS features. Some CSS properties like color are mapped to functionality in the backing UI framework. Sizing, positioning, and general layout is handled by the Yoga layout engine. This intersection gives web developers a familiar experience when working in React Native, but also behaves similarly enough to web to enable code-sharing, as supported by projects like react-native-web.

There are gaps in what React Native supports, with “CSS Features” being the [#1 request](https://github.com/react-native-community/discussions-and-proposals/discussions/528?sort=top#discussioncomment-3886000) in a recent survey to the community over the state of React Native. This intertwines with asks for us to invest in Yoga, which has been [inactive](https://github.com/facebook/yoga/issues/1151). Flexbox support in React Native has had limitations akin to targeting [Internet Explorer 9](http://caniuse.com/?search=column-gap). We also hear a desire for more consistent behavior of existing behaviors with web. This document outlines a set of problems, strategies, and solutions, for building out the intertwined problems of layout conformance, and layout parity.

## Where are the differences?

Differences between layout in RN vs web can roughly be categorized into:

1. **Algorithm Differences:** Given the same styles, normalized for defaults, React Native may still produce a different layout from web browsers. An example of this is a Yoga flag `UseLegacyStretchBehaviour`, where for backwards compatibility, React Native today intentionally diverges from web spec.
2. **Different default behavior:** A new box in react-native is given different [default style properties](https://reactnative.dev/docs/flexbox#flex) than a box on web. E.g. items flow in a column by default instead of a row. Standalone Yoga allows controlling some of these via a `UseWebDefaults` configuration flag, and existing frameworks like `react-native-web` bridge the gap by setting React Native’s default [styles](https://github.com/necolas/react-native-web/blob/dee258ae02188befca4192ea502d4fbebc478341/packages/react-native-web/src/exports/View/index.js#L157) on each View.
3. **Capabilities:** Browser-makers continually add support for new layout capabilities to CSS. This includes new layout systems like CSS grid, but also continual [revisions](https://www.w3.org/TR/css-align-3/) standards which form the basis of Flexbox (e.g. for row-gap and column-gap). 

Where new capabilities are additive, changing default behavior, or existing layout algorithms would affect the way existing components renders. Even if the new behavior is correct, we must be conscious to preserve compatibility with the vast ecosystem of existing React Native components.

## Strict layout

Fixing algorithm behaviors has often caused unintended changes when performed at the scale of large applications. This requires the ability to control layout behavior in a more confined context. A flexible solution for this problem is the introduction of a `<StrictLayout>` component which allows opting its children into more-correct layout behaviors.

```jsx
import {StrictLayout} from 'react-native';

<StrictLayout>
  {...}
</StrictLayout>
```

This style of component has downsides. React encourages encapsulation, where individual components don’t have to reason about the tree above or around them. If `StrictLayout` were to radically shift behavior, it would require component authors to reason about more state outside of their component and negatively impact developer experience. For this reason `StrictLayout` **does not** change the default styles of a component tree. It instead is focused on algorithm correctness, and will only cause changes where components were previously relying on behavior divergent from browsers. After StrictMode is [stable](#staging), we will start flagging to developers where their component rendering is reliant on out-of-spec behavior, and attempt to offer [actionable guidance](#migration-hints).

### Usage

Existing apps or shared components targeting both web and mobile can safely get wins from globally opting into stricter layout algorithms, since any existing layout must render correctly on web. A `<StrictLayout>` component allows opting a component subtree into a continually growing [set of changes](#specific-w3c-conformance-issues) to align closer to web algorithms:

```jsx
// Shared surfaces/components can be authored with consistent rendering
<XPlatApp>
  <StrictLayout>
    {...}
  </StrictLayout>
</XPlatApp>
```

### Staging

There is more than a single conformance change that we would like to make, and we are likely to continually discover new places where we diverge from browsers. This means strict layout may continually evolve, and continue to make new breaking changes. This poses a risk to the community if we were to release `StrictLayout` before we have integrated a full set of changes, since someone who adopts strict layout may need to battle repeated breaking changes.

We should instead treat `StrictLayout` as an unstable API, then coalesce the changes to an eventual point where it becomes official.

```jsx
// Treat StrictLayout as unstable until we have a fuller set of breaks
import {unstable_StrictLayout as StrictLayout} from 'react-native';
```

### Implementation

`StrictLayout` and `ClassicLayout` would internally change the `YGConfig` used to layout individual Yoga nodes. The most valuable implementation is in Fabric, where this code is shared between all platforms. A custom shadow node would have access to all child Yoga nodes, so a single `RCTLayoutConfig` ShadowNode may traverse its full set of children to change Yoga node configuration before the root is laid out.

Where Fabric is potentially further out (e.g. out-of-tree platforms), this set of changes could still be implemented, but the implementation would need to be per-platform.

## Handling different default styles

Changing existing default styles would be an incredibly disruptive change, breaking compatibility with all existing RN components, and making developers need to reason about two sets of rules within the same set of primitives. A separate approach is to define defaults in a higher level component, which allows local reasoning.

```jsx
// Defaults to flexDirection: 'column'
<View />
```
```jsx
// Defaults to flexDirection: 'row'
<Div /> 
```

This direct approach of defining this translation in JS has potential to tangibly increased memory usage. ShadowNodes are pay-for-play in terms of number of properties, and Yoga stylesheets [may be in the future](#mitigating-bloat-to-heap-usage). This could possibly be mitigated as part of the `RCTLayoutConfig` internal interface.

## Fixing the ecosystem

Projects already sharing code with web can safely enable `StrictLayout` at their top-level to opt into a continuing set of fixes. Projects not previously targeting the web need some amount of intervention to safely adopt W3C compliant behaviors behaviors. Strict layout would then fragment the ecosystem if it didn't also offer benefits for existing projects only targeting Android and iOS.

We should instead lean towards a future where components are shareable with web by default, while quarantining components which asume previous behaviors. Apart from better code-sharing, moving the bulk of the ecosystem towards tolerating W3C behavior unlocks new opportunities for [layout engine usage](#betting-on-yoga-alternative-choices). We must take steps to create a critical mass of usage in currently active projects. We can do this by making it easy to fix code which depends on non-conforming layout, and to provide tangible benefits for doing so.

### Classic layout

An app otherwise using strict layout may want to opt specific component trees into classic layout behavior. This can be useful both as a tool for incremental migration (starting by disabling strict layout where it causes differences), cases where code may be unable to be changed (e.g. third-party components), or as a tool to unblock fixing [new algorithm issues](#staging). `<StrictLayout>` then needs a negatinig counterpart `<ClassicLayout>` to allows opting out of a configurable subset of changes (especially useful for large migrations). 

```jsx
// Force fully classic rendering and disable migration hints
<ClassicLayout>
  <CruftView />
<ClassicLayout>
```
```jsx
// Disable a single behavior change and its migration hints
<ClassicLayout quirks=['stale-basis-measurements']>
  <CruftView /> 
</ClassicLayout>
```

### Traceable styles

It is not possible to statically reason about whether a given style will render differently under strict layout. We can instead rely on runtime feedback from the underlying layout engine. We can add an API to Yoga like `YGNodeAffectedByQuirk()`  to query if the engine made choices based on legacy behavior. E.g. on branching to `UseLegacyStretchBehaviour` we can calculate whether the different behavior will lead to a visible difference, and cache that on the yoga node.

The Yoga Node attached to a Shadow Node is far enough away from the JS source defining a style that we need a method to correlate the two. This can be solved by a Babel transform, which inserts a correlation ID into each style, forwarded to the underlying Yoga Node. This ID can be mapped in the bundle to source location. This has a negative effect on bundle size, but should be possible to add to development mode bundles, as a debugging feature.

The below shows a structure the babel transform could replicate:

```jsx
// Original stylesheet
const styles = StyleSheet.create({
  widget: {
    flex: 1,
  },
});

// After babel transform
const styles = StyleSheet.create({
  widget: StyleSheet.compose(
    {
      flex: 1,
    },
    {_creationOrigin: 123},
  ),
});

// Style calculated by function
<View style={calculateStyle()} />;

// After babel transform
<View style={StyleSheet.compose(calculateStyle(), {_applyOrigin: 456})} />;
```

### Migration hints

Traceable styles let us point to what makes styles acts differently under strict layout. React Native uses LogBox warnings for providing guising information to developers, but we must take care not to make them spammy or inactionable. This can be mitigated by:

1. Measuring the occurrence of these warnings silently internally, before rolling out (they should be relatively rare)
2. Disable the warnings if a user explicitly opts into classic layout (such as for abandoned third-party components)

> **⚠️ Warning**
>
> A component was laid out with behavior incompatible with W3C Flexbox. See more at https://reactnative.dev/docs/strict-layout.
>
> Issue: React Native stretched your component along the main axis.
> Fix: Add `flexGrow: 1` to your style if this was intentional or enable `<StrictLayout>` to opt into W3C behavior (no stretching).
>
> View: Widget.js:123<br>
> Style: Widget.js:60

### Restriction to conformant trees

Future layout capabilities may depend on using an engine without the quirks of Yoga. We can enable this future flexibility, and encourage migration (avoid splitting the ecosystem) by requiring StrictLayout to use the new capabilities we add.

> **❌ Error**
>
> `display: 'grid'` is only supported when using strict layout mode. See more at https://reactnative.dev/docs/strict-layout.
>
> View: Widget.js:123<br>
> Style: Widget.js:60

## Betting on Yoga (Alternative Choices)

React Native, and much of the rest of the OSS ecosystem rely on Yoga. Yoga has been in maintenance for the past few years, with the originally owning team not engaging with the community over the last few years. Yoga has had contributors who file issues, and have submit PRs fixing conformance issues (e.g. fixing stale measurements), and adding whole new features (e.g. `gap`/`rowGap`/`columnGap`).

Beyond adding new capabilities directly, we have a long road to enable a technical and social environment which allows contributions to Yoga. We need to fixup the tools used by OSS to build changes against Yoga, and be meaningfully responsive to incoming changes and issues. This is a huge undertaking, that we think is worth it not just for React Native, but the broader community using Yoga. This was compared against several alternatives:

### Yoga 2 (flexlayout)

Yoga 2/flexlayout (not to be confused with a popular set of Yoga bindings also named FlexLayout) is a currently closed source spiritual successor to Yoga, developed by Yoga’s owning team. It is undergoing active rollout within Meta, being used internally for [ComponentKit](https://componentkit.org/), and soon [Litho](https://fblitho.com/). It is derived from Yoga, but offers a simplified Modern C++ codebase, and a programming model which lets the consuming application define its own node structure (Data can be embedded into the ShadowNode instead of a separate Yoga Node tree). Aligning to it would allow unifying resources between teams, which is usually a positive.

Despite these advantages, we’ve chosen not to prioritize migration to Yoga 2. Its future roadmap and maintenance is uncertain as is its role in OSS. It has not shown performance benefits over Yoga, and does not currently implement new features. It is still being stabilized, and is not yet at full parity which Yoga (e.g. as of writing, the `flex` shorthand is unsupported). Some of the goals behind Yoga 2 conflict with the goals of React Native. Yoga 2 was built for environments with a high sensitivity to binary size, where we are more interested in adding new complexity, negatively impacting that goal. Yoga 2 does not support code which assumes the bugs and legacy flags of Yoga 1, and we would need to implement changes in the engine to support the current RN ecosystem. Migration in RN would itself be a very large effort blocking enablement of the innovations we would like. We will continue to reevaluate usage of Yoga 2 as it matures, and our ecosystem becomes more compliant, but should not block improving React Native and our ecosystem in the meantime.

### Browser engines

It is tempting to try to reuse an already developed styling engine from a browser. Major advantages are conformance and capabilities, but these are tied to internals of each browser, and add a large footprint and complexity that is undesirable. We would also need to build in Yoga bug compatibility. It would effectively entail rebuilding a new layout engine, heavily forked from a one-time snapshot of a browser engine.

### Taffy

[Taffy](https://github.com/DioxusLabs/taffy) is a Flexbox layout engine written in Rust, authored by Emil Sjölander, a main contributor of Yoga. It is seeing active development under new hands, and has some capabilities that Yoga doesn’t (e.g. it recently added `align-items: 'space-evenly'`). It does not have bug compatibility with the existing RN ecosystem, its rollout/production usage is unknown, and being owned outside of Meta limits our ability to direct the engine in ways which are targeted towards RN’s use-cases.

### Platform-specific controls

Platform-holders like Microsoft and Google have looked at (or have) implemented their own FlexBox controls, such as `google/FlexboxLayout`. For cases where these controls may be made in-box, we get to punt on a lot of complexity, and make React Native apps smaller. Not all platforms have native flexbox controls though, they do not match the set of capabilities we want, and they do not have bug compatibility or Meta ownership.

## Validating conformance

We know about some subset of conformance issues, but there are likely to be more, and as we add new capabilities, we must be systematic in ensuring a correct implementation. Browsermakers share a set of tests for validating conformance called [WPT (Web Platform Tests)](https://github.com/web-platform-tests/wpt/tree/master/css). These have already been leveraged by React Native for pointer events, by porting the HTML to React Native. An alternative to porting the code may be to rely on Yoga’s existing capabilities of generating unit test derived from how Chrome lays out HTML fixtures.

## Specific W3C conformance issues

While we are likely to discover more conformance issues (and there are pending issues and PRs on GitHub that should be examined), the below list targets some of the issues that have been observed by framework authors, and individual developers:

1. **`UseLegacyStretchBehaviour`:** A Yoga configuration flag set by react-native for bug compatibility, which may incorrectly stretch an items flex-basis, even when told not to grow. It has been a source of multiple reports of different behavior between browsers and RN, and does not require additional implementation within the layout engine.
2. **Stale flex basis:** Yoga does not correctly invalidate flex basis after it changes. A fix for this was added, gated behind `YGExperimentalWebFlexBasis`, but there are concerns over its correctness. A long running issue with text truncation in a large Meta application was root-caused to this issue. An existing [pull request](https://github.com/facebook/yoga/pull/1017) may offer a fix.
3. **Box-sizing:** While not strictly related to Yoga, react-native offers a box-sizing similar, but reportedly not quite in line with `box-sizing: 'border-box'`. More investigation is needed into the exact behavior.
4. **Two-pass flexible item resolution:** Yoga [deviates](https://github.com/facebook/yoga/blob/5dd33acc912711c599250e3533b07c732de503ee/yoga/Yoga.cpp#L2365) from the W3C spec on how the lengths of flexible items are resolved, implementing a simpler/more performant algorithm.
5. **Content-based minimum size:** Yoga intentionally [does not](https://github.com/facebook/yoga/blob/5dd33acc912711c599250e3533b07c732de503ee/yoga/Yoga.cpp#L2662-L2665) assign a minimum size to flexible items in deviation from the flexbox spec, for performance reasons.
6. **Min/Max in flexible items:** At least according to [in-code documentation](https://github.com/facebook/yoga/blob/5dd33acc912711c599250e3533b07c732de503ee/yoga/Yoga.cpp#L2666-L2667), Yoga does not respect min/max size constraints on the main axis for flexible items. Because of a lack of support this is unlikely to affect existing RN code, but affects code ported from web.
7. **Differences in circular dependencies:** Yoga handles styles differently from browsers when a parent is sized based on content, and a child is sized based on its parent. This was guessed to be the cause of layout differences when porting code using Yoga previously to web (Meta employees see [this link](https://fb.workplace.com/groups/yogalayout/posts/7192992174106050/?comment_id=7197040323701235)).

## Synchronous Layout

While not a W3C conformance issue, React Native doesn’t expose the DOMs methods of synchronous layout, which bleeds into differences in the React programming model. React has both “effects“ and ”layout effects”. On the web, effects (e.g. code in `useEffect`) are executed after the component has been laid out and painted. Layout effects are executed directly after DOM mutation, and block painting. In React Native, both effects and layout effects are executed in parallel with layout and painting. This limits the ability for components to synchronously measure components, or programmatically respond to layout changes, without relying on first committing them to screen. Layout computation is not thread affined today (it can happen off the UI thread). A solution for this is to allow synchronously running layout on the JS thread. This was previously attempted, but had (possibly mitigable) performance concerns when measured.

## Choosing the capabilities to build

React Native does not implement the full set of browser CSS features. Apart from higher level concepts like stylesheets, selectors, and cascading (which can largely be supplanted by CSS-in-JS solutions shared between RN and browser), React Native doesn’t implement all style properties.

It’s clear there is a desire for more capabilities, but there is a question of which should be added. React Native should enable skills to be transferable, offer a stellar developer experience, and enable substantial code-sharing (while being leaner than a browser). A natural conflict is whether to prioritize building for compatibility (existing usages), or for developer experience. [Adoption of conformant layout](#restriction-to-conformant-trees) is dependent on building out features developers would like to use, in their existing React Native projects. 

With the initial focus of building for developer experience, instead of compatibility, we need input on what the community cares about. We can derive a lot of this from asking directly, but should be opportunistic in our choices, accounting for implementation complexity.

## Mitigating bloat to heap usage

Yoga internally represents a stylesheet as a packed structure of all possible properties. This structure is already 192 bytes, scaling with each additional Yoga Node. As we add more styles, this could expand. Doubling the number of styles on a tree of 20,000 nodes would increase memory consumption by almost 4MB, a huge amount for some of the devices React Native targets. We should measure, and potentially switch YGStyle to a data structure which is pay-for-play in number of properties. This shift would be performance sensitive, but theoretically quick data structures exist (e.g. a bitfield header with a small-vector for larger values). 

## Example capabilities

The following is a non-exhaustive list of capabilities where Yoga lags browsers. We should consult specs used by yoga (CSS Box Alignment, CSS Box Sizing, CSS Flexbox, etc) for a more exhaustive list.

1. **`display: grid`:** CSS Grid is highly used in responsive designs on the modern web, allowing some layouts which are hard or not possible to express with Flexbox. It is one of the most requested additions, but has one of the highest implementation cost of the listed examples.
2. **`display: inline/block/table/ruby`:** Yoga treats text as inline, but other nodes act as `display: flex`. The listed display modes could help compatibility, but Flexbox has superseded much of their usage.
3. **`display: contents`:** Useful for composing boxes which should not alter child layout (e.g. embedding event handlers)`.`
4. **`position: fixed/sticky`:** React native does not support `position: fixed` or `position: sticky`. React Native’s closest API is `stickyHeaderIndices`, which allows top-level ScrollView items to be sticky.
5. **`box-sizing`:** React Native does not support specifying box-sizing, and anecdotally is closest to but does not fully conform to `border-box`. implementing box-sizing allows new modes which may be conformant.
6. **`align-content: space-evenly`:** Yoga does not supported `space-evenly` as a value for flexbox alignment.
7. **`place-content`:** Shorthand for `align-content` and `justify-content`. Trivial to add but has a memory const without [sparse stylesheets](#mitigating-bloat-to-heap-usage).
8. **`inset:`** Shortland for top/left/bottom/right (similar considerations to `place-contents` as a shorthand property)
9. **Viewport based units `(vmin, vh, vw, etc)`:** React Native + Yoga does not support units derived from viewport/root node size.
10. **Font based units `(em, rem, etc)`:** React Native + Yoga does not support units derived from the size of the current font. Configurable `fontSize`  in React Native is currently confined to text components, and will not cascade. Its pixel value would then be scaled by `fontSize` if set on a Text component or by system default font if unset on Text, or on a View.
11. **Percentages in more places:** Some existing functionality like gap is implemented with support for pixels, but are missing support for percentage.
12. **`min-content`:** As part of not supporting content-based minimum sizing, the `min-content` keyword is not supported.

## Other styling gaps

This document focuses specifically on layout, but there are other gaps in styling between React Native and browsers. Common examples are in `box-shadow` ,  `filter`, `transform`,  `mask`, and `clip-path`. We should evaluate the complexity of implementing each, for the underlying native platforms which they would interact with.


