---
title: ArrayBuffer support in TurboModules
author:
  - Kamil Paradowski <kamil.paradowski@callstack.com>
date: 10.10.2025
---

# RFC0000: ArrayBuffer support in TurboModules

## Summary

This RFC outlines adding first-class `ArrayBuffer` support to TurboModules to enable zero-copy binary data exchange between JavaScript and native modules.

Codegen and TurboModules currently lack efficient binary data handling, forcing developers to rely on workarounds, such as using the `CodegenTypes.UnsafeObject` type or inefficient conversion to `base64`, when working with binary-heavy use cases. This feature addresses performance bottlenecks in data-intensive applications. By providing first-class `ArrayBuffer` support, developers will be able to seamlessly pass large binary payloads between JavaScript and native code without the overhead of data serialization and copying. This enhancement will unlock new possibilities for creating responsive, high-performance React Native applications across various domains including media processing, real-time communication, and AI/ML applications that require efficient handling of tensors and model data.

## Basic example

Developers should be able to define a TypeScript spec as follows:

```ts
export interface Spec extends TurboModule {
  getBuffer(): ArrayBuffer;
  processBuffer(buffer: ArrayBuffer): void;
}
```

## Motivation

TurboModules currently lack a first-class way to represent `ArrayBuffer` end-to-end in Codegen, which forces developers to rely on copies, ad-hoc platform bridges, global helpers, or external libraries. This hurts performance for binary-heavy use cases such as media data or ML tensors, and it increases implementation complexity. The expected outcome is a cross-platform contract that lets JS and native pass binary data with minimal copying. Codegen should be able to generate working code for the `ArrayBuffer` type on every platform.

For example, several important use cases are currently difficult to implement efficiently while working with TurboModules:

- **Real-time media streaming**: A native video decoder could stream frames directly to a JavaScript-based player component. Without zero-copy `ArrayBuffer`s, each frame would need to be copied, leading to significant performance overhead and potential frame drops.
- **Machine Learning**: On-device ML models often require passing large tensors between native inference engines and JS. Copying this data can be a major bottleneck, especially for real-time applications like video analysis.
- **High-performance networking**: Applications that handle large binary payloads over WebSockets or other protocols (e.g., financial data streams, real-time gaming) may be forced into inefficient data conversion, which adds CPU and memory pressure.

By providing a first-class `ArrayBuffer` type support to TurboModules, this RFC will unblock these and other performance-sensitive areas, making it possible to develop faster more efficient applications for React Native.

## Detailed design

This section contains a description of the design and related topics. It's split by topics and affected packages/areas. In the expandable sections, extensive code snippets can be found. In the final implementation, each area should come with appropriate unit tests.

### Memory ownership

When passing an `ArrayBuffer` to native code, it should always be treated as "borrowed" or "non-owning": JS owns the ArrayBuffer's memory and the JS GC is responsible for freeing it. Native code should access the passed memory only for the duration of the synchronous call.

The same rules apply when passing buffers from native to JS: native code remains the owner of the allocated memory and may expose zero-copy buffers to JS.

### Threading

`ArrayBuffer` implementations are not thread-safe; if multiple threads simultaneously read from or write to an `ArrayBuffer`, race conditions can occur. To prevent this, developers must ensure that an `ArrayBuffer` is not accessed concurrently from different threads, for example by making sure that JavaScript thread does not modify the `ArrayBuffer` while native code is working on it.

### Types

To address this gap in Codegen, we first need to specify the corresponding native type for every language used by TurboModules. The type should have the following traits:

- Provide access to the raw memory of the buffer
- Provide the length of the buffer
- Provide a way to mutate/modify the buffer contents
- Provide a zero-copy construction path without extra allocations
- Not take ownership of the underlying memory block

Results can be found in the table below.

| Language    | Type                  |
| ----------- | --------------------- |
| JavaScript  | `ArrayBuffer`         |
| C++         | `jsi::ArrayBuffer`    |
| Java        | `java.nio.ByteBuffer` |
| Objective-C | `NSMutableData`       |

We need types that provide zero-copy semantics and clear ownership of the underlying data. Fortunately, we can reuse existing native types instead of implementing custom ones. Below is a breakdown by platform.

#### C++

The C++ case requires the fewest changes and least discussion when choosing the native type. JSI already has first-class support for ArrayBuffers. The JSI `jsi::Object` can hold an ArrayBuffer instance. `jsi::Object` provides `ArrayBuffer getArrayBuffer(Runtime& runtime)` and `bool isArrayBuffer(Runtime& runtime)`, which make it easy to convert objects to buffers. `jsi::ArrayBuffer` provides access to internal data and its size. Moreover, this class inherits from `jsi::Object`, can be implicitly converted to its parent class, and is intended to be constructed in a non-copying way (using move semantics or by passing a shared pointer to `jsi::MutableBuffer`).

However, to maintain consistency with other types, we should add a helper `ArrayBuffer asArrayBuffer(Runtime& runtime)` to the `jsi::Object` class.

#### Java

Java provides a class that matches our needs: `java.nio.ByteBuffer`. It offers the required functionality and can be created via `ByteBuffer.allocateDirect()` or `NewDirectByteBuffer()` to wrap existing memory without extra allocations — unlike a `byte[]` alternative. It also provides access to the raw data pointer, individual bytes, and its length.

#### Objective-C

For Objective-C we chose `NSMutableData` for the same reasons described above. It allows creating an object that wraps existing memory and clearly defines ownership of the memory block.

### Zero-copy conversion

Below you can find required changes and example zero-copy conversion implementations for all platforms.

#### C++

No changes are needed in JSI. Its in-place support for `ArrayBuffer` already provides the necessary in-place conversion functionality.

#### Java

Java class `java.nio.ByteBuffer` can be constructed using JNI function `jobject NewDirectByteBuffer(JNIEnv* env, void* address, jlong capacity)`. This function returns a direct instance of direct `java.nio.ByteBuffer` referring to the block of memory starting at the memory address `address` and extending `capacity` bytes. What is important, direct buffers don't deallocate the memory on destruction, what is desired in our case since JS is responsible for that.

<details>
<summary>Example conversion implementation</summary>

`packages/react-native/ReactCommon/react/nativemodule/core/platform/android/ReactCommon/JavaTurboModule.cpp`:

```cpp
  // jsi::ArrayBuffer -> java.nio.ByteBuffer
  if (type == "Ljava/nio/ByteBuffer;") {
    if (!(arg->isObject() && arg->getObject(rt).isArrayBuffer(rt))) {
      throw JavaTurboModuleArgumentConversionException(
          "ArrayBuffer", argIndex, methodName, arg, &rt);
    }
    auto arrayBuffer = arg->asObject(rt).asArrayBuffer(rt);
    auto len = arrayBuffer.size(rt);
    auto data = arrayBuffer.data(rt);
    auto directBuffer = env->NewDirectByteBuffer(
        static_cast<void*>(data), static_cast<jlong>(len));
    jarg->l = makeGlobalIfNecessary(directBuffer);
    continue;
  }
  ...
  // java.nio.ByteBuffer -> jsi::ArrayBuffer
  case ArrayBufferKind: {
    auto returnObject =
        (jobject)env->CallObjectMethodA(instance, methodID, jargs.data());
    checkJNIErrorForMethodCall();

    TMPL::syncMethodCallExecutionEnd(moduleName, methodName);
    TMPL::syncMethodCallReturnConversionStart(moduleName, methodName);

    auto returnValue = jsi::Value::null();
    if (returnObject != nullptr) {
      auto jResult = jni::adopt_local(returnObject);
      auto byteBuffer = jResult.get();

      struct ByteArrayMutableBuffer : jsi::MutableBuffer {
        public:
           ByteArrayMutableBuffer(uint8_t* data, size_t size)
               : _data{data}, _size{size} {}

           uint8_t* data() override {
             return _data;
           }
           size_t size() const override {
             return _size;
           }

        private:
           uint8_t* _data{};
           size_t _size{};
      };

      auto size = env->GetDirectBufferCapacity(byteBuffer);
      auto data = (uint8_t*)env->GetDirectBufferAddress(byteBuffer);
      auto mutableBuffer =
          std::make_shared<ByteArrayMutableBuffer>(data, size);
      auto arrayBuffer = jsi::ArrayBuffer{runtime, mutableBuffer};
      auto obj = jsi::Value{runtime, arrayBuffer};
      returnValue = std::move(obj);
    }

    TMPL::syncMethodCallReturnConversionEnd(moduleName, methodName);
    TMPL::syncMethodCallEnd(moduleName, methodName);
    return returnValue;
  }
```

> [!NOTE]
> In the final implementation temporary structure `ByteArrayMutableBuffer` can be moved outside of the function.

</details>

#### Objective-C

In Objective-C, we need conversions between `facebook::jsi::ArrayBuffer` and `NSMutableData`. Creating an instance of `NSMutableData` wrapping the memory of the JS `ArrayBuffer` without taking ownership of it can be achieved by calling the `dataWithBytesNoCopy` function with `freeWhenDone` set to `NO`.

<details>
<summary>Example conversion implementation</summary>

`packages/react-native/ReactCommon/react/nativemodule/core/platform/ios/ReactCommon/RCTTurboModule.mm`:

```objc
// NSData -> jsi::ArrayBuffer
static jsi::ArrayBuffer convertNSDataToJSIArrayBuffer(jsi::Runtime &runtime, NSData *value)
{
	auto length = [value length];
	auto buffer = (uint8_t*)[value bytes];

	class NSDataMutableBuffer : public facebook::jsi::MutableBuffer {
	 public:
		NSDataMutableBuffer(uint8_t* data, size_t size)
			: _data{data}, _size{size} {}

		uint8_t* data() override {
			return _data;
		}
		size_t size() const override {
			return _size;
		}

	 private:
		uint8_t* _data{};
		size_t _size{};
	};

	auto mutableBuffer = std::make_shared<NSDataMutableBuffer>(buffer, length);
	auto arrayBuffer = jsi::ArrayBuffer(runtime, mutableBuffer);
	return arrayBuffer;
}
...
// jsi::ArrayBuffer -> NSData
static NSMutableData *convertJSIArrayBufferToNSData(
		jsi::Runtime &runtime,
		const jsi::ArrayBuffer &value,
		const std::shared_ptr<CallInvoker> &jsInvoker,
		BOOL useNSNull)
{
	auto length = value.size(runtime);
	auto buffer = value.data(runtime);

	NSMutableData* data = [NSMutableData dataWithBytesNoCopy:buffer length:length freeWhenDone:NO];

	return data;
}
```

> [!NOTE]
> In the final implementation temporary structure `NSDataMutableBuffer` can be moved outside of the function.

</details>

### Usage

The end goal of this RFC is to enable developers to use the `ArrayBuffer` type in spec files, use Codegen to generate native interfaces and glue code, and implement TurboModule functions that accept or return `ArrayBuffer`.

#### TypeScript

```typescript
export interface Spec extends TurboModule {
  getBuffer(): ArrayBuffer;
  processBuffer(buffer: ArrayBuffer): void;
}
```

#### C++

```cpp
class JSI_EXPORT NativeBufferCxxSpecJSI : public TurboModule {
protected:
  NativeBufferCxxSpecJSI(std::shared_ptr<CallInvoker> jsInvoker);

public:
  virtual jsi::ArrayBuffer getBuffer(jsi::Runtime &rt) = 0;
  virtual jsi::Value generateAsyncBuffer(jsi::Runtime &rt) = 0;
  virtual jsi::String processBuffer(jsi::Runtime &rt, jsi::ArrayBuffer buffer) = 0;
};
```

#### Java

```java
public abstract class NativeBufferSpec extends ReactContextBaseJavaModule implements TurboModule {
  ...
  @ReactMethod(isBlockingSynchronousMethod = true)
  @DoNotStrip
  public abstract java.nio.ByteBuffer getBuffer();

  @ReactMethod(isBlockingSynchronousMethod = true)
  @DoNotStrip
  public abstract String processBuffer(java.nio.ByteBuffer buffer);
}

```

#### Objective-C

```objc
@protocol NativeBufferSpec <RCTBridgeModule, RCTTurboModule>

- (NSMutableData *)getBuffer;
- (NSString *)processBuffer:(NSMutableData *)buffer;

@end
```

### Scope of `ArrayBuffer` support for Codegen

So far this RFC mentioned support for `ArrayBuffer` only for a specific range of use cases. As can be noticed in the section above, these are:

- Function output argument (`getBuffer`)
- Function input argument (`processBuffer`)

However, Codegen provides more functionality and broader support for types that can be shared across platforms. Below you can find the list (probably not full) of them:

- Promises

```ts
export interface Spec extends TurboModule {
  getBuffer(): ArrayBuffer;
  getAsyncBuffer(): Promise<ArrayBuffer>;
}
```

- Structs

```ts
export type ObjectStruct = {
  a: number;
  b: string;
  c?: ?string;
};
export type BufferStruct = {
  text: string;
  value: ArrayBuffer;
};
```

- Constants (special case of Structs)
- Unions (can use Struct underneath)

```ts
export type UnionFloat = 1.44 | 2.88 | 5.76;
export type UnionObject = { value: number } | { buffer: ArrayBuffer };
```

- Events

```ts
export interface Spec extends TurboModule {
  onEvent: CodegenTypes.EventEmitter<string>;
  onData: CodegenTypes.EventEmitter<ArrayBuffer>;
}
```

As illustrated, the `ArrayBuffer` type can be used in many contexts; a simple input/output parameter is only one of them. Handling each separately will be required to provide full and consistent support across all platforms. In order to pinpoint potential problems we investigated the topic at this stage. Here is a breakdown of the results by platform.

#### C++

It looks like the current implementation of `AsyncEventEmitter` requires an argument type to be copyable. That is not the case for `jsi::ArrayBuffer`. This will probably require deeper investigation and perhaps more complex changes to the bridging part of the `ReactCommon` library.

#### Java

While in C++ and Objective-C data can be easily shared between JS and Native, Java stores the data as a `folly::dynamic` map on the Native side. The `folly` library has support for data buffers (class `IOBuf`). This means that JS buffers can be stored on the Native side, but its implementation will be more challenging. Moreover, classes responsible for storing variables, such as `NativeMap` or `NatviveArray`, have a rich inheritance tree and are widely used across the JNI files. Adding storage for buffers to them will require changes to a large number of `ReactAndroid` JNI and Java/Kotlin files. These changes are required to add support for e.g. Promises or Structs.

#### Objective-C

No obstacles were found for this platform.

#### Conclusion

To conclude, supporting only simple function input/output parameters is straightforward, but extending that support to cover all Codegen functionalities across every platform is significantly more complex. However, integrating Promises and asynchronous operations may be crucial for supporting binary-heavy use cases. This leads to the question:

> [!IMPORTANT]
> Should this RFC focus on introducing only the basic and most valuable synchronous support for ArrayBuffer, or should it aim for full coverage of all possible Codegen use cases, including asynchronous operations, despite the higher complexity and the impact on more files (especially on Android)?

## Drawbacks

This RFC should be treated more as an extension of the current set of supported types in Codegen rather than a brand new feature. There are no significant costs associated with it; it seems like a natural evolution of TurboModules.

## Alternatives

I considered other native types. For Java, `byte[]` was considered but it doesn't meet the construction requirements. For Objective-C, `NSData` was rejected because it is immutable.

## Adoption strategy

Adoption will be non-breaking. The Codegen changes introduce a new type and generate platform glue that is backwards-compatible with existing TurboModules. Developers who are currently passing large buffers inefficiently (e.g., using `UnsafeObject` or `base64`) will need to manually update their code after this feature is implemented.

## How we teach this

This feature can be taught in a few different ways:

- Add to the docs a type/argument conversion table showing which native type each supported JavaScript type (including `ArrayBuffer`) maps to. Right now there is no easy way to view the list of supported types in Codegen; a similar table exists only in the Legacy Architecture section.
- Create a dedicated doc page explaining the importance of memory management, lifetime and threading of `ArrayBuffer` in TurboModules.
- Expand the `rn-tester` application to use the new type in one of its example modules.

## Unresolved questions

Open questions/topics for discussion:

1.  Thread-safety of the `ArrayBuffer` - do we need to implement any additional synchronization mechanism to make this change thread-safe?
2.  Should we introduce two kinds of buffers, mutable and read-only, in order to improve DX and solve potential concerns about thread-safety?
3.  Should this RFC focus on introducing only the basic and most valuable synchronous support for ArrayBuffer, or should it aim for full coverage of all possible Codegen use cases, including asynchronous operations, despite the higher complexity and the impact on more files (especially on Android)?
