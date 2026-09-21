---
title: Binary Props — phasing folly::dynamic out of Fabric's Android update pipeline
author:
- Bartłomiej Błoniarz
date: 2026-07-07
---

# RFC1013: Binary Props — phasing folly::dynamic out of Fabric's Android update pipeline

## Summary

**Target: less allocations and JNI calls when moving prop updates between c++ and JVM**

On Android, each prop update is stored in the `folly::dynamic` `rawProps` member of the C++ `Props` struct. The pipeline passes this dynamic object straight to the platform through JNI. Thus the values in the other fields of `Props` are mostly ignored.

Props 2.0, now in development, changes this approach. Codegen makes diff functions that compare the C++ structs when mounting occurs. Only the diff goes to the mounting layer. This gives a useful property: to apply many revisions, diff only the first revision and the last revision (the Pull Model).

In the current implementation, each diff is a `folly::dynamic` object. Each diff object must go to the mounting layer. Java represents it as a `ReadableNativeMap`. This is an fbjni hybrid object that keeps the `folly::dynamic` on the C++ side. The representation is lazy.

To materialize the map, Java makes two bulk JNI calls: `importKeys()` and `importValues()`. A third call, `importTypes()`, occurs when types are queried. The result puts all keys in a `String[]` and all values in a boxed `Object[]`, in a Java `HashMap`. This causes many separate memory allocations. It also causes multiple JNI calls — O(number of updates).

This RFC proposes a different design. Allocate one buffer up front. Use codegen to make diff functions that write into this buffer. Pass the buffer memory to the JVM once. Generated readers then forward the updates to the view managers.

On an entry-level phone, the measured cost of this boundary drops from **~257 µs to ~36 µs per animated view per frame** (full results in *Measured performance*). These numbers come from a prototype in the best case. We must evaluate them again at each step of the project.

## Basic example

Currently a custom component is defined like that, a js spec:

```js
type NativeProps = Readonly<{
  ...ViewProps,
  speed?: WithDefault<Float, 1.0>,
  glowColor?: ColorValue,
}>;
export default codegenNativeComponent<NativeProps>('RNTFancyGlowView');
```
and the ViewManager, which implements the generated `RNTFancyGlowViewManagerInterface<T>` (codegen derives the name from the component name and also emits the matching `RNTFancyGlowViewManagerDelegate`):

```kotlin
// author-written — identical before and after this RFC
class FancyGlowViewManager : SimpleViewManager<FancyGlowView>(),
    RNTFancyGlowViewManagerInterface<FancyGlowView> {
  override fun setSpeed(view: FancyGlowView, value: Float) { view.speed = value }
  override fun setGlowColor(view: FancyGlowView, value: Int?) { view.glowColor = value }
}
```

**Today, with Props 2.0**, codegen creates the `getDiffProps` function that returns a `folly::dynamic` that is then wrapped in a `ReadableNativeMap` in `FabricMountingManager.cpp` (`getProps()`) and written into the mount transaction. The delegate's `setProperty` method is generated to unwrap these values in JVM and forward them to the user-defined ViewManager.

```cpp
// generated C++ (today)
folly::dynamic RNTFancyGlowViewProps::getDiffProps(const Props* prevProps) const {
  const auto* oldProps = static_cast<const RNTFancyGlowViewProps*>(prevProps);
  folly::dynamic result = HostPlatformViewProps::getDiffProps(prevProps);
  if (speed != oldProps->speed) { result["speed"] = speed; }              // heap alloc, string key
  if (glowColor != oldProps->glowColor) { result["glowColor"] = *glowColor; }
  return result;  // → one ReadableNativeMap per view, bulk-imported over JNI with every value boxed
}
```

The diff reaches the delegate through three hops:

```cpp
// FabricMountingManager.cpp — getProps(): the only place the diff crosses to Java
auto diff = newProps->getDiffProps(oldProps);
return ReadableNativeMap::newObjectCxxArgs(std::move(diff));  // one Java-visible map per view
```

```kotlin
// IntBufferBatchMountItem.kt → SurfaceMountingManager.kt
INSTRUCTION_UPDATE_PROPS ->
    surfaceMountingManager.updateProps(intBuffer[i++], objBuffer[j++] as ReadableMap)
// SurfaceMountingManager.updateProps():
viewState.currentProps = ReactStylesDiffMap(props)
checkNotNull(viewState.viewManager).updateProperties(view, viewState.currentProps)
```

```java
// ViewManager.java — updateProperties(): the JNI import happens here, per view, per update
Iterator<Map.Entry<String, Object>> iterator = props.internal_backingMap().getEntryIterator();
while (iterator.hasNext()) {
  Map.Entry<String, Object> entry = iterator.next();
  delegate.setProperty(viewToUpdate, entry.getKey(), entry.getValue());
}
```

```java
// generated Java delegate (today)
public void setProperty(T view, String propName, Object value) {
  switch (propName) {                                       // string match
    case "speed":
      mViewManager.setSpeed(view, value == null ? 1f : ((Double) value).floatValue());  // boxed
      break;
    ...
  }
}
```

**Instead**, codegen can generate `getBinaryDiffProps` that uses a writer to put its updates into a shared buffer. On the java side we generate a `setPropertyBinary` method that forwards the values from the buffer to the user-defined viewManager.

```cpp
// generated C++ (this RFC)
void RNTFancyGlowViewProps::getBinaryDiffProps(
    BinaryPropsRecordWriter& writer, const Props* prevProps) const {
  HostPlatformViewProps::getBinaryDiffProps(writer, prevProps);  // base View range, ids 1–511
  if (speed != oldProps->speed) { writer.float32(512, speed); }
  if (glowColor != oldProps->glowColor) { writer.color(513, glowColor); }
}
```

```java
// generated Java delegate (this RFC)
public void setPropertyBinary(T view, int propId, BinaryPropsReader reader) {
  switch (propId) {
    case 512: mViewManager.setSpeed(view, reader.wasNull() ? 1f : reader.readFloat()); break;
    case 513: mViewManager.setGlowColor(view, reader.wasNull() ? null : (Integer) reader.readInt()); break;
    default: super.setPropertyBinary(view, propId, reader);   // base View props
  }
}
```

Records from all views land in one buffer per mount transaction and cross JNI once.

## Motivation

### The current solution doesn't scale well

For every updated view, the current pipeline:

1. builds a `folly::dynamic` diff object in C++ — a heap allocation per prop, keyed by strings;
2. wraps it into one `ReadableNativeMap` **per view** (`getProps()` in `FabricMountingManager.cpp`) — a JNI hybrid object allocation; the map is lazy, and the contents are pulled into Java by **bulk JNI calls** (`importKeys()` + `importValues()` in `ReadableNativeMap.kt`);
3. walks the map in Java — string-matching each prop name in a generated switch, receiving every value as a boxed `Object` (`Double`, `Integer`).

The two structural problems are **many small allocations** and **per-view JNI crossings to materialize the maps** — both scale per prop × per view × per frame (this is especially a problem for animations).

### Before and after

| per view per frame | map path (today) | binary path (this RFC) |
|---|---|---|
| serialize the diff (C++) | build `folly::dynamic` — allocations, string keys | append fixed records to one shared buffer |
| cross the boundary | 1 JNI map object + bulk import calls (every value boxed), per view | one buffer crossing **per transaction** |
| dispatch (Java) | string-match names, unbox `Object`s, parse transform strings | id-switch, unboxed reads|
| **measured boundary cost** | **~257 µs** | **~36 µs** |

Whole-frame and React-commit prototype results are in *Measured performance*.

## Detailed design

### Prerequisite: Props 2.0

Binary emission starts from field-level diffs. Props 2.0 already provides them. Codegen generates `getDiffProps` / `getDiffPropsImplementationTarget` for every component. The ~8 hand-written core Props classes implement the same pair. With the flags active, `Props::rawProps` stays empty for conforming components.

The Props 2.0 flag set contains three flags:

- `enablePropsUpdateReconciliationAndroid` — the mount computes the update with `getDiffProps`. It compares the last mounted shadow node with the newly committed shadow node.
- `enableAccumulatedUpdatesInRawPropsAndroid` (Props 1.5) — merges the previous `rawProps` into the new ones. A dynamic diff against the last mounted tree is then complete. This is the fallback for non-conforming components.
- `enableExclusivePropsUpdateAndroid` — skips the Props 1.5 `rawProps` accumulation fully when Props 2.0 covers the component. This is what leaves `Props::rawProps` empty.

The gate operates per component. `getDiffPropsImplementationTarget()` must match the registered component name (`ConcreteComponentDescriptor.h`). Hand-written props classes that have not implemented the pair keep the fallback.

### The protocol

We read the batch by iteration only. Random access is not necessary. Thus we use a flat representation of the data. A u16 id identifies each prop. The data follows the id in an inlined form.

The id space needs attention. Custom components can derive only from the base view. The codegen schema's `ExtendsPropsShape` admits exactly one value, `ReactNativeCoreViewProps` (`CodegenSchema.js`). Thus we can divide the id space into two segments:

```text
ids   1…511   base ViewProps — core-owned
ids 512…      component-local
```

Different native components can reuse the ids over 512. The mounting layer forwards the records of each view to the generated `ViewManagerDelegate` for that view's component. The delegate lookup goes through the view's `ViewManager`, exactly as the string path does. Thus a component-range id never reaches the switch of a different component.

### Covering every prop type

Every prop type has a binary encoding — the design goal is that we can move the data with a single JNI call without per view allocations. Below we show what could be the shape of the generated code for various edge cases in the js component specs.

```js
type GlowStop = Readonly<{ offset: Float, color: ColorValue }>;
type NativeProps = Readonly<{
  ...ViewProps,                                       // transform, backgroundColor … (base range)
  speed?: WithDefault<Float, 1.0>,                    // scalar, id 512
  glowColor?: ColorValue,                             // optional without default, id 513
  halo?: Readonly<{ radius: Float, tint?: ColorValue }>,  // nested object, id 514
  stops?: ReadonlyArray<GlowStop>,                    // array of structs, id 515
  meta?: UnsafeMixed,                                 // Mixed, id 516
}>;
```

**1. Scalars and optionals.** A changed scalar is one record; an optional without a default emits a null flag when cleared.

```cpp
// differ
if (speed != oldProps->speed) writer.float32(512, speed);
if (glowColor != oldProps->glowColor)
  glowColor ? writer.color(513, *glowColor) : writer.null(513);
```
```java
// reader
case 512: mViewManager.setSpeed(view, reader.wasNull() ? 1f : reader.readFloat()); break;
case 513: mViewManager.setGlowColor(view, reader.wasNull() ? null : (Integer) reader.readInt()); break;
```

**2. Transforms.** Today the transform crosses the boundary as the ops array (`[{translateX: 5}, {rotateZ: 0.785}, …]`). `getDiffProps` re-serializes the typed `TransformOperation`s back into dynamics. Java stores the ops in a view tag. Java resolves the ops into a 4×4 matrix with `TransformHelper.processTransform`. This step resolves `%` translates against the current view size.

Java then decomposes the matrix (`MatrixMathHelper.decomposeMatrix`). It applies translate, rotate, and scale to the view. When the layout size of the view changes, Java runs all these steps again from the stored ops (`onLayoutChange` in `BaseViewManager`). Three encodings are possible:

1. *Serialize the ops* — write records of `(opType, value, unit)`. C++ already holds the transform normalized as `TransformOperation{type, ValueUnit×3}`, with angles pre-parsed to radians. Thus no side must parse strings. The Java pipeline after that stays byte-for-byte the same, including the layout-time `%` re-resolution. This option is fully faithful. But it keeps the per-frame Java matrix math.

2. *Resolve to a 4×4 matrix in C++ and ship 16 floats.* Use `BaseViewProps::resolveTransform`, the same call that iOS makes on every mount. Java already has a fast path for exactly this shape. A 16-number transform array skips parsing and goes straight to `decomposeMatrix`. Caveat: we cannot resolve a baked matrix again when only the layout changes. `%` translates and `transformOrigin` would go stale. Those cases would also need a props update on each layout change.

3. *Decompose in C++ and ship the translate, rotate, and scale components directly.* The reference implementation does this for the canonical `translate* rotate? scale?` shape (one 8×f32 record). It falls back to the map path for `%`, skew, perspective, `matrix`, and multi-axis rotations. This decode is the leanest — no Java matrix math at all. But this option is very focused on Android. Other platforms that could benefit from a binary path could not reuse it.

**3. Nested objects.** In the spec they have a fixed, declared shape (`halo` above). Unlike Mixed, codegen generates a typed C++ struct for them, and that is the member type:

```cpp
// generated Props (today)
struct RNTFancyGlowViewHaloStruct {
  Float radius{0.0};
  SharedColor tint{};
  bool operator==(const RNTFancyGlowViewHaloStruct&) const = default;
};
RNTFancyGlowViewHaloStruct halo{};
```

With Props 2.0 the diff is at prop granularity: the compare is field-wise, but on any change the whole object is serialized back into a dynamic:

```cpp
// generated getDiffProps (today)
if (halo != oldProps->halo) { result["halo"] = toDynamic(halo); }
```

and the setter receives the complete value as a `ReadableMap`:

```java
// generated Java delegate (today)
case "halo":
  mViewManager.setHalo(view, (ReadableMap) value);
  break;
```

We must keep this contract: a record always carries the whole new value of the prop, never a partial value. The setter replaces the previous value. Managers rely on this contract. They keep the received object and use it after the setter returns. `BaseViewManager` stores `transform` and `filter` in view tags; it re-reads the `transform` tag on layout changes and the `filter` tag in `onAfterUpdateTransaction`. `ReactScrollView` keeps the last `contentOffset` map and compares the next map against it.

The binary encoding writes the fields in spec order. A presence bit marks each optional member. Managers retain the value, as shown above, so the reader copies the span of the record. The reader wraps the copy in a lazy `ReadableMap`. A generated descriptor supplies the field names. If a consumer needs real random access, it can build an ordinary Java map from the serialized data.

Managers also compare retained maps against the next value. Thus `BinaryReadableMap.equals` must compare content and must match `ReadableNativeMap`. The same view can receive one type from creation and the other type from an update.

```cpp
// differ
if (halo != oldProps->halo) {
  if (!halo) { writer.null(514); return; }
  auto obj = writer.beginObject(514);
  obj.float32(halo->radius);
  halo->tint ? obj.color(*halo->tint) : obj.absent();     // presence bit
}
```
```java
// reader — HALO is the generated field-name/kind/offset descriptor
static final BinaryStructDescriptor HALO = descriptor().f32("radius").optionalColor("tint");
case 514: mViewManager.setHalo(view,
    reader.wasNull() ? null : new BinaryReadableMap(reader.readOwnedSpan(), HALO)); break;
```

The setter keeps its `setHalo(view, ReadableMap)` signature, so nothing changes for component authors. Typed Java classes could be generated later as a per-component opt-in; that changes the author API, so not during the migration.

**4. Arrays and arrays of structs.** A count, then the elements back to back — nesting recurses.

```cpp
// differ
if (stops != oldProps->stops) {
  auto arr = writer.beginArray(515, stops.size());
  for (const auto& s : stops) { arr.float32(s.offset); arr.color(s.color); }
}
```
```java
// reader
static final BinaryStructDescriptor GLOW_STOP = descriptor().f32("offset").color("color");
case 515: mViewManager.setStops(view,
    reader.wasNull() ? null : new BinaryReadableArray(reader.readOwnedSpan(), GLOW_STOP)); break;
```

Arrays use the same owned-span copy as nested objects, for the same reason.

**5. Mixed / `UnsafeMixed`** — the one type with no static schema. In the spec it is the escape hatch for "any JSON value" - the shape is not declared and can change between renders:

```js
<FancyGlowView meta={{glow: 'outer', levels: [1, 2, 3]}} />
<FancyGlowView meta={42} />   // also legal
```

Libraries use it to keep props loosely typed - react-native-svg wraps `fill`, `stroke` and `font` in `UnsafeMixed`, and those are commonly animated.

At parse time the whole JS value is walked and materialized into a `folly::dynamic` (`jsi::dynamicFromValue`), and that is also the type of the generated member - there is no static type that could hold it:

```cpp
// generated Props (today)
folly::dynamic meta{};
```

With Props 2.0 there is no typed field to diff, so the generated diff deep-compares the two dynamics and on change copies the whole value into the diff object:

```cpp
// generated getDiffProps (today)
if (meta != oldProps->meta) { result["meta"] = meta; }   // deep compare + deep copy
```

which then crosses as part of the view's `ReadableNativeMap`, and the setter receives a `Dynamic` (Mixed can be a scalar, not just a map):

```java
// generated Java delegate (today)
case "meta":
  mViewManager.setMeta(view, new DynamicFromObject(value));
  break;
```

For the binary path there are several approaches we can test:

1. **Inline self-describing records.** Write records of tag + key + value, length-prefixed, into the shared buffer. The Java side wraps the span in a lazy `ReadableMap`/`Dynamic` cursor. The cursor decodes on access. This uses the existing writer and reader, with zero extra JNI objects. The record must carry an explicit byte length. This can replace `dynamic` because we do not update the value after serialization. We only read it and overwrite it.

```cpp
// differ — serialize the dynamic into the buffer
if (meta != oldProps->meta) writer.tagged(516, meta);
```
```java
// reader — a lazy cursor over the tagged bytes; keys and types are in the record, not a schema
case 516: mViewManager.setMeta(view,
    reader.wasNull() ? null : new BinaryDynamicMap(reader.readOwnedSpan())); break;
```

2. **Reuse the current `folly::dynamic` → `ReadableNativeMap` path for this prop only.** The transaction already carries an object array next to the buffer. We can continue to ship the dynamic there as a `ReadableNativeMap`. The record in the buffer only says that the value must be popped from that array.

```cpp
// differ — the record only marks the prop; the value ships in the object array
if (meta != oldProps->meta) writer.other(516, meta);
```
```java
// reader — pop the next object from the transaction's object array
case 516: mViewManager.setMeta(view,
    reader.wasNull() ? null : new DynamicFromObject(objBuffer[j++])); break;
```

3. **Use the existing MapBuffer.** MapBuffer assumes that both sides compile against the same schema. Its keys are `uint16_t` constants agreed at build time. The text pipeline uses it this way: `TA_KEY_FONT_SIZE = 4` in `TextAttributeProps.kt` mirrors the C++ writer. But Mixed is exactly the prop with no schema. The string keys would have to travel in the buffer. MapBuffer cannot represent that. It also cannot represent null values or heterogeneous arrays. Also, `MapBufferBuilder::build()` owns its allocation and crosses JNI as one hybrid object per map.

### Dispatch and base-prop overrides

The string path dispatches as follows today. The `setProperty` switch of the generated delegate covers the component's own props. An unmatched prop falls through `super.setProperty` to `BaseViewManagerDelegate`. There, a `when` over the `ViewProps` names ends in a **virtual call on the concrete ViewManager** (`mViewManager.setBackgroundColor(…)`).

A component spec can redeclare a base prop name. Then codegen emits a case for it in the derived switch. This case **shadows** the base case. Exactly one setter runs per prop.

The binary path must reproduce both behaviors. Base-range ids fall through to `setPropertyBinary` on the base delegate. A redeclared base prop gets a component-range id, so it continues to shadow. The debug dual-encoding check (see *Adoption strategy*) catches any divergence.

```kotlin
// BaseViewManagerDelegate — the base range, the same virtual calls as the string switch
override fun setPropertyBinary(view: T, propId: Int, reader: BinaryPropsReader) {
  when (propId) {
    BACKGROUND_COLOR -> mViewManager.setBackgroundColor(view, reader.readInt())
    OPACITY -> mViewManager.setOpacity(view, reader.readFloat())
    ...
  }
}
```

A derived delegate shows all three behaviors in one switch — its own props, a redeclared base prop, and delegation. Here the spec redeclares `opacity`, so codegen assigns it a component-range id and the derived differ emits that id instead of the base one:

```java
// generated Java delegate (this RFC) — spec redeclares base `opacity`
public void setPropertyBinary(T view, int propId, BinaryPropsReader reader) {
  switch (propId) {
    case 512: mViewManager.setSpeed(view, reader.wasNull() ? 1f : reader.readFloat()); break;
    case 513: mViewManager.setGlowColor(view, reader.wasNull() ? null : (Integer) reader.readInt()); break;
    case 514: mViewManager.setOpacity(view, reader.wasNull() ? 1f : reader.readFloat()); break;  // shadows the base case
    default: super.setPropertyBinary(view, propId, reader);   // base range 1–511 → BaseViewManagerDelegate
  }
}
```

### View creation

`getDiffProps(nullptr)` diffs against static defaults, so creation and preallocation reuse the same protocol. Layout props never travel this pipeline — they ship as layout metrics.

## Measured performance

Every number in this document is a device measurement from a working reference implementation of the full pipeline, taken on an entry-level phone (OPPO A16, Helio G35), release build. The workload is 200 accessibility-heavy views animating `{opacity, transform}`; every run is screencap-verified for actual on-screen motion. The benchmark harness ships with the reference implementation, so results stay re-measurable.

| scenario (200 animated views, per frame) | map path | binary path |
|---|---|---|
| animation, synchronous route | 64.3 ms | **17.1 ms** |
| animation, commit route (Props 2.0 field diffs on both sides) | 51.7 ms | 27.8 ms |
| React-driven re-render, same screen | 82.9 ms | 59.1 ms |

Per-view, the mount-boundary slice goes from ~257 µs (per-view map construction + Java consume) to ~36 µs (binary decode + typed setters). The remaining commit-route time is dominated by tree diffing and the platform setters themselves; those are follow-up territory outside this RFC's scope.

## Adoption strategy

**Safety mechanism.** In debug builds, run both encodings for each mount. Decode the binary records back to a map. Assert equality with the `getDiffProps` output. Every debug session then becomes a parity test. Props 2.0 shipped with the same cross-validation pattern.

**Transitional fallback (removed at the end).** This applies only during the rollout. Some prop types have no landed encoding yet. Some components are not regenerated yet. Each such case routes the whole view to the existing map path for that commit. The generated diff emits `writer.other()` for each value it cannot encode yet. The mount boundary then rolls those records back:

```cpp
props->getBinaryDiffProps(writer, oldProps);
if (writer.failed()) { writer.rollback(); return false; }   // not yet encodable → map path
```

## Drawbacks

- **Two encodings during migration** - could be alleviated by an `#ifdef` 
- **Public API changes** - we try to keep them minimal within this RFC, but there would be some changes in updateState that currently reads the props as is

## Alternatives

- **Reuse MapBuffer as the props protocol**: MapBuffer stores a type tag per entry and keeps its keys sorted, because its reader is generic and needs random access. Our reader is generated from the same schema as the writer, so the prop id alone tells it the record layout, and the records are consumed once, in order - random access is not needed.
- **Optimize the map path** (pooling, lazier maps): doesn't really address the core problem of many small allocations and many JNI calls, and is a lot of work for a small gain
- **Introduce a fast path for animated props**: this is something that reanimated already has for non-layout props, but it is not a general solution, and increases the scope of RN update flows
- **Do nothing**: every animation library keeps reimplementing an unsupported fast path against internal APIs.

## How we teach this

Component authors: nothing changes — regenerating picks up the fast path; new guidance is one rule, "props read during state updates belong in setter-maintained view fields." Core contributors get the protocol spec doc (record layouts, id-space rules, the coverage plan above).

## Open questions

### Scope

This RFC focuses mostly on the `getDiffProps` path. That path is part of the React Native mounting layer. There, the binary diffs are calculated as the result of a new commit. But the Animation Backend has two paths that apply updates. One of them bypasses the ShadowTree and would benefit greatly from the binary diffs. The two paths are:

```c++
  for (auto& [surfaceId, updates] : surfaceUpdates) {
    if (updates.hasLayoutUpdates) {
      commitUpdates(surfaceId, updates);
    } else {
      synchronouslyUpdateProps(updates.propsMap);
    }
  }
```

The first path applies updates through a commit to the ShadowTree. The second path applies them to the platform itself. Ideally, an Animation Backend user defines one batch of updates in this format. The user passes the batch to the backend. The backend then applies the updates to the ShadowTree, or directly to the platform.

Thus the protocol must be deserializable on both sides. The JVM side applies updates on the platform. The C++ side applies updates to the ShadowTree. The Animation Backend already has a `PropsBuilder` API whose typed setter surface is currently unused. We could adopt it for this purpose. At this stage we could focus mainly on the `getDiffProps` path, but the largest wins for animations could come from the path that bypasses the ShadowTree.

An alternative plan phases out the `synchronouslyUpdateProps` path. All updates then go through the ShadowTree. In that case, this RFC can focus on `getDiffProps` and continue to apply updates to the ShadowTree in the old way (which could be improved with a future RFC). This is a win for correctness, because the ShadowTree is the source of truth. But it could be worse for performance. To make this difference irrelevant, we must optimize ShadowTree cloning and diffing.

### The `Mixed` record format

We must select one approach from *Covering every prop type*. The fallback cannot be permanent, because the end goal deletes the map path. Also, libraries such as react-native-svg hide frequently animated props behind `UnsafeMixed`. Thus the likely end state is the inline self-describing record. Its layout is open: how the keys are stored, and whether the Java cursor implements `Dynamic` directly.
