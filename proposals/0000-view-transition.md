---
title: ViewTransition support in React Native
author:
- Zeya Peng
date: 2026-07-10
---

# RFC0000: ViewTransition support in React Native

## Summary

This RFC proposes adding support for React's [`<ViewTransition>`](https://react.dev/reference/react/ViewTransition) component to React Native. `<ViewTransition>` is a React API that lets you animate elements that update inside a transition. On the web it delegates to the browser's View Transition API; this proposal brings the same programming model to React Native by integrating the React reconciler with the Fabric renderer.

## Basic example

**Example 1 — Shared transition**

A `<ViewTransition>` with a `name` in one render is paired with a `<ViewTransition>` using the same `name` in the next render. When the state changes inside `startTransition`, the runtime automatically animates from the old element's layout to the new one.

```tsx
const [showFirstView, setShowFirstView] = useState(true);

const fadeInUp = keyframes({
  from: { opacity: 0, transform: 'translateY(-30px)' },
  to: { opacity: 1, transform: 'translateY(0px)' },
});

// render
showFirstView && (
  <ViewTransition name="some-view-transition-name">
    <View style={{ /* ... */ }} />
  </ViewTransition>
);

!showFirstView && (
  <ViewTransition
    name="some-view-transition-name"
    share={viewTransitionClass({
      new: {
        animationName: fadeInUp,
        animationDuration: '1s',
      },
    })}>
    <View style={{ /* ... */ }} />
  </ViewTransition>
);

// Triggering the transition:
<Button
  onPress={() => {
    startTransition(() => {
      setShowFirstView(false); // triggers the shared transition
    });
  }}
/>
```

This should behave the same if `<Activity mode={showFirstView ? "hidden" : "visible"}>` is used to hide/show the content instead of conditionally rendering it.

**Example 2 — Enter/exit transition**

When the whole subtree (including the `<ViewTransition>` itself) mounts or unmounts inside a transition, the runtime drives an enter or exit animation.

```tsx
const [shouldShow, setShouldShow] = useState(false);

// render
{shouldShow ? (
  <ViewTransition enter={/* ... */} exit={/* ... */}>
    <View>Something to show</View>
  </ViewTransition>
) : null}

<Button
  title="reveal content"
  onPress={() => {
    startTransition(() => {
      setShouldShow(true);
    });
  }}
/>
```

**Example 3 — Enter/exit via `<Activity>` (hide/show instead of mount/unmount)**

The same enter/exit animation fires when the content is hidden/shown with `<Activity>` rather than conditionally rendered. Unlike mount/unmount, an `Activity` switching between `visible` and `hidden` keeps the subtree alive but still picks up the transition heuristics — the `<ViewTransition>` must be the top-level child directly under `<Activity>`.

```tsx
const [shouldShow, setShouldShow] = useState(false);

// render
<Activity mode={shouldShow ? 'visible' : 'hidden'}>
  <ViewTransition enter={/* ... */} exit={/* ... */}>
    <View>Something to show</View>
  </ViewTransition>
</Activity>

<Button
  title="reveal content"
  onPress={() => {
    startTransition(() => {
      setShouldShow(s => !s);
    });
  }}
/>
```

## Motivation

React Native already offers built-in animation via `Animated` and `LayoutAnimation`, and Reanimated is the most popular open-source library filling the remaining gaps. Even so, `<ViewTransition>` covers transition use cases that none of these serve well today.

| Capability | Animated | LayoutAnimation | Reanimated | ViewTransition |
| --- | --- | --- | --- | --- |
| Declarative animation config (time/curve) on style props; imperative start/stop | ✅ | | ✅ | |
| Subscribe to native event (e.g. scroll) value | ✅ | | ✅ | |
| Animate a view from layout A to B; runtime interpolates automatically | | 🚧 unstable | ✅ | ✅ |
| **Shared transition** — animate view A (layout X) → view B (layout Y); runtime interpolates automatically | | | 🚧 experimental | ✅ |
| Runtime automatically detects and drives enter/exit transitions | depends on useEffects | 🚧 unstable | ✅ | ✅ |

`<ViewTransition>` uniquely provides the last two rows in a general, cross-tree way — where the others are single-view, unstable, or experimental.

**What does `<ViewTransition>` add to the current React Native ecosystem?**

Beyond the capability gaps in the table above (built-in libraries and Reanimated), it adds:

- **A first-class way to create transition animations in React** that takes advantage of React's modern feature and existing mechanism in Reconciler — including integration with **Suspense and Activity**. Because it lives in React, you can animate from a Suspense fallback to its content when ready, and an `Activity` switching between visible/hidden picks up the same transition heuristics as mount/unmount.
- **Predictable enter/exit detection.** Before `<ViewTransition>`, reliably running an animation when a component shows or hides was hard, and neither of the two available approaches — JS effects or native mutation interception — detects it well.
  - *`useEffect`-based detection is fragile:*
    - `useEffect` with an empty dependency list runs only on first mount; if you navigate to a route stacked on top and come back, the component was never unmounted, so no enter animation fires.
    - With real dependencies, the callback can fire unexpectedly when an unrelated dependency changes, or when a hidden `Activity` becomes alive.
    - Passive effects don't wait for native mount, so you can see a flash of the new component before the animation starts.
  - *Native mutation interception (the New Architecture approach used by RN LayoutAnimation & Reanimated) can't reliably detect enter/exit either:*
    - It sees only anonymous native-view Create/Remove mutations, so it can't tell that a Remove here and a Create there are "the same element" — which is why plain enter/exit can't do `layoutId`-style pairing without a separate tag registry.
    - It animates any registered view mutation regardless of cause, rather than being gated on Transitions — so it can't distinguish a deliberate enter/exit from an incidental insert/delete.
    - Switching an `<Activity>` between visible/hidden does **not** generate Create/Remove mutations at all, so a mutation-interception approach simply can't observe it.
  - Because React drives `<ViewTransition>` from the reconciler — where it sees both trees, gates on Transitions, and understands `Activity` — it detects enter/exit at the *component boundary* with **correct timing**: it knows the boundary between the old and new tree and invokes callbacks before native mount, avoiding the visual "blink" that both user-space approaches struggle with, including on `Activity` visibility changes.

Because React sees both the old and new tree, controls commit timing, and understands Suspense/Activity, this capability is better expressed as a React-integrated component rather than reinvented in a library. See [Alternatives](#alternatives) for a detailed comparison with Reanimated.

## Detailed design

The high-level strategy is to reuse what the React reconciler already solves (detecting enter/exit/update/share, ordering the transition phases, batching) and implement the renderer-side configuration hooks in Fabric. On the web these hooks map onto the browser's View Transition API; in React Native they drive Fabric and the animation runtime directly.

### Integration with the React reconciler

React drives view transitions through a set of renderer config functions. The DOM renderer implements them by assigning `view-transition-name` CSS to elements (the browser then snapshots and animates them). The Fabric renderer implements the equivalent behavior:

- **Marking participants.** When a transition starts, update (via UIManager) a registry in Fabric noting which nodes participate in the next transition, with what `viewTransitionClass`, and the inferred transition type (enter/exit/share/update). This mirrors the DOM renderer adding/removing `view-transition-name`.
- **Measuring.** The renderer measures layout and captures old/new snapshots at the point React calls the measurement config function.
- **Detecting transition type.** React determines enter/exit/update/share by diffing the trees; the Fabric renderer only needs to honor those signals. (See "Reference: React DOM renderer behavior" below for the detection rules React uses.)

### Driving the transition animation

#### Shared transition

This is the uniquely new problem for React Native, since the browser provides it for free on the web. Several implementation options were considered.

A key consideration is that, eventually, the animating subtree should be **reparented to an overlay layer** (a shared, screen-global coordinate space above both screens) for the duration of the transition — this is what the recommended options build toward. Three problems make in-place animation insufficient for real shared transitions:

- **Clipping.** The destination view starts inside its final parent (a card, a list cell, a `ScrollView` with `overflow: hidden`). Animating it from the source position means escaping the parent's bounds — impossible without disabling clipping up the entire ancestor chain.
- **Z-order.** Mid-flight, the element must draw above both screens (during a push the outgoing screen is still visible underneath). In place, it's stuck beneath whatever siblings/screens sit above it in the hierarchy.
- **Two hierarchies at once.** During navigation the old and new screens are both live; neither screen's local coordinate space can host an element that spans both. An overlay provides a shared, screen-global coordinate space for the element to travel through.

The options:

- **Option 1 — Drive animation imperatively on the new node, in place (recommended short term).** Compute the delta (relative to the viewport origin) from the old to the new layout, then drive the animation with the shared animation backend, using a JS-side placeholder to avoid pushing around the neighbors of the animating component. Because Yoga re-runs layout during the transition, the whole animating subtree stays correctly laid out — important for subtrees containing text. Reuses the Fabric animation driver and doesn't require CSS-animation support as a prerequisite; developers can customize with a declarative animation API. Downside: the node is animated in place rather than lifted onto an overlay, so it hits the clipping/z-order/two-hierarchy limits above (no new pseudo element, not lifted above the document as on the web) and is less web-compatible — acceptable for simpler same-screen cases but not full cross-screen shared transitions.

  Illustrative example: the `onShare`/`onUpdate` callback receives the `ViewTransition` instance for the new node, and we imperatively start an animation that tweens its real layout props (`left`/`top`/`width`/`height`) from the old rect to the new rect. Because it animates the real layout props (rather than a `transform`), Yoga re-runs flexbox during the transition and the persistent children reflow to their new positions:

  ```tsx
  const [shrunk, setShrunk] = useState(false);

  <ViewTransition name="parent-a" onShare={(instance)=>{
    // Start the animation imperatively on the new node, in place. Ensure the animation
    // animates layout props so Yoga re-runs layout during the transition
    Animated.sharedTransition(instance, {
      duration: 400,
    }).start();
  }}>
    <View style={[styles.parent, { width: shrunk ? 176 : 320 }]}>
      {CHILD_COLORS.map((color, i) => (
        <View key={i} style={[styles.child, { backgroundColor: color }]} />
      ))}
    </View>
  </ViewTransition>

  // The state change must run in a transition for ViewTransition to fire.
  <Pressable onPress={() => startTransition(() => setShrunk(s => !s))}>
    <Text>{shrunk ? 'Grow A' : 'Shrink A'}</Text>
  </Pressable>
  ```
- **Option 2 — Drive animation and subtree reparenting with Shadow Tree commits (recommended long term).** Reparent the new ShadowNode to the React root during the transition (Fabric does not allow this today) and restore it when the transition ends; host views reparent accordingly. Layout stays correct because Yoga recomputes during the transition. Downside: requires new Fabric capability to reparent a ShadowNode.
- **Option 3 — Drive animation on the host platform, using existing native transition solutions.** Delegate the shared transition to the platform's own APIs — e.g. even leveraging Jetpack Compose's `SharedTransitionLayout` with `sharedElement`/`sharedBounds` on Android, or UIKit custom (interactive) transitions on iOS. Conceptually these solutions reparent the participating host view under the React root's host view, animates old→new relative to root, then moves it back, which is closest to the web and avoids React-tree/Shadow-tree divergence. Downsides: no observability of the intermediate state, no cross-platform solution — because each platform's transition API consumes its own animation spec, we'd have to re-implement the animation-config parsing/interpolation infra per platform instead of reusing the cross-platform one; and these native solutions may not composite/synchronize well with React Native's existing animations (Fabric-driven `Animated`/shared backend), leading to desync between platform and Fabric animations running at the same time.
- **Option 4 — Animate a copied ShadowView with per-frame mutations.** Copy the participating ShadowView under the React root, hide the real new view, and push interpolated mutations each frame (like `LayoutAnimation`), then clean up on completion. Downsides: mounts two host views for one node (expensive), only animates the immediate child rather than the whole subtree, and transition state is not observable to Fabric. (This is close to how Reanimated's shared transition works today.)

#### Enter transition

```tsx
const [shouldShow, setShouldShow] = useState(false);

const flyIn = keyframes({
  from: { opacity: 0, transform: [{ translateY: -100 }] },
  to: { opacity: 1, transform: [{ translateY: 0 }] },
});

{shouldShow ? (
  <ViewTransition
    name="hello"
    enter={{ default: { animationDuration: 400, animationName: flyIn } }}>
    <View {/* ... */} />
    <Button title="reveal content" onPress={() => startTransition(() => setShouldShow(true))} />
  </ViewTransition>
) : null}
```

Enter transitions don't animate between two viewport-relative locations, so likely no reparenting is needed and the animation can run on the live representation of the new node using the Fabric animation driver. This requires infra to parse the from/to animation spec, compute intermediate frames, and build the animation graph (all of which `Animated`'s JS handles for the `Animated` case).

#### Exit transition

An exit transition operates on a snapshot image of the old node — an "old pseudo element" that outlives the old node, since the component is already unmounted by the time the animation starts. Because we can't reuse an existing on-screen ShadowNode/host view, we must create a representation for the old pseudo element. Options:

- **Option 1 — Create a dedicated shadow node for the old pseudo element within Fabric (recommended).** Animate with the existing Fabric driver; node stays synchronized across Fabric and the host platform. Downside: these are extra shadow nodes not visible to React, so their lifecycle must be handled carefully (Fabric's commit hooks help). Proposed lifecycle:
  - *Transition about to start:* create the old-pseudo-element shadow node when the reconciler detects an exiting component, and capture/cache the old snapshot on the host platform.
  - *Transition starts:* commit the nodes by appending them under the root node in a commit hook.
  - *Transition ready:* draw the cached snapshot when the pseudo element's host view mounts; drop the cached snapshot once drawn (or if no target host view is found) so it doesn't carry over.
  - *Transition complete:* clear the pseudo-element registry so nodes aren't re-appended on subsequent updates; the shadow nodes and host views are removed on a later React update. (Browsers clean up as soon as the animation ends; this is slightly later.)
- **Option 2 — Create a React fiber node for the old pseudo element.** Lifecycle managed by React; synchronized across React/Fabric/host. Downside: the DOM renderer has no such fiber, so this commits a Fabric-only concept into the reconciler.
- **Option 3 — Create a host view with no associated shadow node.** No shadow node that doesn't map to a React component. Downsides mirror the platform-driven shared-transition option (coordination with the mounting transaction, no intermediate-state observability, no cross-platform solution — re-implementing the animation-config parsing/interpolation infra per platform — and composability concerns).

### Coordinating overlapping transitions

If a transition fires while another is still animating, it should be **queued** until the in-flight animation ends, so the user sees one animation after another. This is common both when intentionally sequencing transitions (e.g. fade in a route, then fly in a child once that finishes) and when nested components independently include transition effects that shouldn't disrupt each other.

React Native should use the reconciler's signal (the call of the "start view transition" config function) to decide when to queue a pending transition, wait for the current animation to finish before resolving the finish promise and cleaning up, and only then run the next start.

### Styling a transition

```tsx
const customAnimation = {
  from: { opacity: 0, transform: 'translateY(-30px)' },
  to: { opacity: 1, transform: 'translateY(0px)' },
};

const customViewTransitionClass = StyleSheet.viewTransitionClass({
  share: { animationName: customAnimation, animationDuration: '1s' },
});

<ViewTransition name="some-view-transition-name" share={customViewTransitionClass}>
  {/* ... */}
</ViewTransition>

// ...
startTransition(() => {
  addTransitionType('customTransitionType');
  // react update
});
```

A `viewTransitionClass` assigned to `<ViewTransition>` holds the animation spec its immediate children apply. During the mutation phase, the "apply view transition name" config function receives the child ShadowNode and the `viewTransitionClass`. Because `completeSurface` may have already kicked off, the child could otherwise mount without the class applied. Options:

- **Option 1 (recommended) — Run a second Fabric commit that blocks mount** (`mountSynchronously=false`) to apply the `viewTransitionClass` props when the config function is called. Type-safe; the props apply before mount so the animation can pick them up; specs are synced with Fabric (needed for general CSS animation/keyframes support later).
- **Option 2 — Register the `viewTransitionClass` as a dynamic object** and apply it when animating. Guarantees the class arrives before first mount, but is not type-safe and doesn't reuse Fabric's props infra.
- **Option 3 — Use `UIManager.synchronouslyUpdateViewOnUIThread`.** No-op here, because the initial React update hasn't been mounted yet.

### Transition with video

For shared transitions involving video, users may want playback to continue throughout the transition (feasible on mobile). A common approach in nowadays mobile frameworks is to reuse the same underlying player between the old and new video views.

### Reference: React DOM renderer behavior

For alignment, the DOM renderer's order of operations is roughly: lock new async commits → snapshot phase (add `view-transition-name` to potential participants, take snapshots) → `startViewTransition(mutationPhase)` → browser captures before/after and builds pseudo-elements → `await transition.ready` (revert names, invoke `onEnter`/`onExit`/`onUpdate`/`onShare`) → layout-effect phase → `await transition.finished` (passive effects) → unlock new async commits. Transitions are not interruptible by other async commits; a new transition while one is running batches to the last update. Enter/exit require the `<ViewTransition>` (and everything being hidden/shown, including itself) to mount/unmount together; share vs. exit+enter is determined by whether a matching `name` is created before the transition ends or within the same commit. React Native mirrors this model through the Fabric config functions described above.

## Drawbacks

(Why should we _not_ do this?)

- **CSS animation gap.** `<ViewTransition>`'s most web-compatible styling relies on CSS keyframes and transitions, which React Native does not fully support today. The recommended short-term approach avoids requiring this, but true web parity depends on it.
- **Pseudo elements are a new concept in React Native.** There is no existing support for pseudo elements, and reusing `ShadowNode` as-is is imperfect: a pseudo element should be paint-able before layout effects and generally shouldn't re-layout when its layout props are mutated. This requires careful, Fabric-specific lifecycle handling. Moreover, *truly* supporting new pseudo elements the way the web does — copying/reparenting a subtree to another place in the tree — is expensive on mobile. In React's model many components are automatically included in a transition, but not all of them are actually animated, so eagerly materializing a large number of pseudo elements that ultimately do nothing is unnecessarily costly.
- **Parts are feasible in user space.** Styling a transition (names/classes/keyframes) and a basic single-view shared transition can be built without core changes — so the core investment could be justified only for the reconciler-integrated capabilities (component-boundary detection, pre-mount timing, Suspense/Activity) that user space cannot replicate.

## Alternatives

The closest existing solutions are two Reanimated features, both driven by intercepting native-view mutations:

- **Reanimated layout animations** (`entering`/`exiting`/`layout` props) animate a *tagged view* whenever its mount/unmount or frame change is observed in a commit. They are registered and evaluated per view, with no boundary and no cross-view coordination.
- **Reanimated shared element transition** intercepts Fabric mutations to pair elements and inject animation mutations, which works across native navigation, but only animates the container node (the subtree isn't re-laid out), creates a new element by copying the ShadowView, is still experimental with no clear path out, and doesn't naturally extend to router-level transitions.

The deeper distinction is **where detection happens**. React detects transitions at the *component boundary* — it knows the old and new tree — whereas both Reanimated features operate on anonymous native-view mutations (see [Motivation](#motivation) for how this affects enter/exit detection specifically). Two consequences matter most for shared and layout transitions:

- **Pairing needs a tag registry.** Because it sees only anonymous Create/Remove mutations, Reanimated cannot know that a Remove here and a Create there are "the same element"; `layoutId`-style pairing requires a separate tag registry, whereas React knows it from the tree.
- **Per-subtree vs. per-view, with no group coordination.** React's `<ViewTransition>` boundary applies to a whole subtree. Reanimated's layout transition is registered and evaluated per *tagged view*: the `layout` prop animates only that view's own frame (position + size) when Yoga assigns a new one; children are laid out at their final positions inside the parent's animating frame and merely come along visually. To animate a child smoothly you must give the child its own `layout` prop, and there is **no `LayoutGroup`-style coordination** — per a Reanimated maintainer, layout transitions "run independently and don't know of each other," so a parent's layout transition and a child's enter/exit visibly desync (a common complaint). React's reconciler-level model coordinates the whole boundary.
- **When the end layout is known, and subtree reflow.** Because Reanimated reads the end-view layout *after* the commit, children do not automatically reflow to the new layout during the animation — the subtree is stretched/squashed to fit the animating parent frame rather than re-laid out. With `<ViewTransition>`, the end layout can be obtained *before* the animation starts, so Yoga can recompute layout for the whole subtree and children reflow correctly throughout the transition. This matters most for subtrees containing text paragraphs, which must re-wrap rather than scale.

## Adoption strategy

- **Additive and opt-in.** `<ViewTransition>` is a new component; existing apps are unaffected until they use it. This is **not a breaking change**.
- **Staged rollout.** Ship behind an experimental flag, validate with internal products first, then promote through experimental → canary → stable as the API and web-compatibility gaps settle.
- **Web-aligned API.** Because the component mirrors React's web `<ViewTransition>`, developers moving between React DOM and React Native reuse the same mental model, and shared code is easier to keep aligned.
- **Coordination with libraries.** Where relevant, align with Reanimated and adopt the shared animation backend so a single animation driver underpins View Transitions and other animation APIs.

## How we teach this

- **Framing.** Present `<ViewTransition>` as a continuation of the existing React pattern, documented alongside React's own [ViewTransition reference](https://react.dev/reference/react/ViewTransition), emphasizing that transitions are triggered by state updates inside `startTransition`.
- **Terminology.** Reuse the web/React vocabulary — `name`, `share`, `enter`, `exit`, `update`, `viewTransitionClass`, transition types — so knowledge transfers directly from the web.
- **Docs impact.** React Native's animation documentation would gain a new section positioning `<ViewTransition>` relative to `Animated`, `LayoutAnimation`, and Reanimated (the capability table above is a good starting point), plus recipes for shared, enter/exit, and router-level transitions.
- **For existing developers.** Teach it as the reliable replacement for `useEffect`-based enter/exit hacks and hand-rolled shared-element plumbing, highlighting the correct-timing and old/new-node-access guarantees.

## Prototype

A working prototype — an Expo app that puts Reanimated's layout animation side by side with React's `<ViewTransition>` — drives the examples in this proposal. It lives on a dedicated branch: [`viewtransition-demo`](https://github.com/zeyap/discussions-and-proposals/tree/viewtransition-demo/proposals/0000-view-transition-demo/layout-anim-vt-demo).

It demonstrates:

- **Update / layout reflow** *(available)* — the `onUpdate` example under [Shared transition](#shared-transition): a parent shrinks and its flex-wrap children reflow to their new wrapped positions (rather than scaling), showing that the animating subtree re-runs layout during the transition. The demo places this **side by side with the equivalent Reanimated layout animation** so the reflow difference is directly visible.
- **Enter / exit** *(TBD)* — a subtree that mounts/unmounts inside `startTransition` drives an enter or exit animation ([Basic example 2](#basic-example)). Not yet implemented in the demo.
- **Shared transition** *(TBD)* — a `<ViewTransition>` paired by `name` across renders animates from the old element's layout to the new one ([Basic example 1](#basic-example)). Not yet implemented in the demo.

Current status: Android, driven by the Fabric animation runtime. iOS and CSS-animation-driven styling are not yet implemented (see [Unresolved questions](#unresolved-questions)).

## Unresolved questions

- **Should CSS Animation be a prerequisite, or ship after?** We could ship the primitives (view-transition event handlers and pseudo-element rendering) and let developers animate with the library of their choice (`Animated`/Reanimated or others), then add CSS animation/keyframes later. CSS animation may be friendlier for AI-assisted authoring and SSR, but developers often "break the rules" when starting animations imperatively.
- **How should we support a "new pseudo element"?** We believe we *should* support it (lifting the new element above its document, onto an overlay, as on the web) — it's what makes cross-screen shared transitions work (clipping/z-order/two-hierarchy, see [Detailed design](#driving-the-transition-animation)). The question is *how*, given the cost of reparenting/re-mounting on mobile (see [Shared transition](#shared-transition)).
- **Which web disparities must be closed before OSS release?**
  - Web animates a new pseudo element; the recommended RN approach animates the actual new element in place.
  - The parent of a transitioning component isn't pinned to a fixed layout during the transition.
  - View transitions on RN don't currently work across native navigation (on the web, view transitions can work across documents).
- **Platform coverage.** iOS support and the remaining reconciler integration (e.g. the `onUpdate` event) are still to be shipped.

## References

- React `<ViewTransition>`: https://react.dev/reference/react/ViewTransition
- "How does ViewTransition work?": https://react.dev/reference/react/ViewTransition#how-does-viewtransition-work
- Reanimated shared element transition (experimental): https://github.com/software-mansion/react-native-reanimated/pull/7466/
