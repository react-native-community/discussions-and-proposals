---
title: JavaScriptCore-safe source URLs
authors:
- Rob Hogan (@robhogan)
- Moti Zilberman (@motiz88)
date: 2023-05-02
---

# RFC0646: JavaScriptCore-safe source URLs

## Summary

Add support for a query-string-free URL format to bundlers, so that symbolication (etc.) is not broken by JavaScriptCore error stack manipulation introduced in https://github.com/WebKit/WebKit/commit/71985aa1520560154a329e5b54159fff3173cf22 (first appearing in iOS 16.4).

## Basic example

`http://localhost:8080/project/index.bundle//&platform=ios`

Should be supported by Metro (and other bundlers) equivalently and in addition to the current query-string form:

`http://localhost:8080/project/index.bundle?platform=ios`

## Motivation

Recent versions of JavaScriptCore strip query strings from source URLs in stack traces (`Error.prototype.stack`).

To implement features such as Metro's `/symbolicate` endpoint, we must be able to derive (a close approximation of) the source that was executed the client. To do so correctly we need to know the build parameters the client used originally. Those parameters being stripped (and defaults used instead) has manifested as obscure resolution errors for React Native users, on both Expo and the community CLI ([1](https://github.com/facebook/react-native/issues/36794), [2](https://github.com/expo/expo/issues/22008)).

Ideally, for maximal compatibility with debuggers (etc.) that may intercept the raw error stack, the source URLs in the error stack should be sufficient, and should be valid URLs, without any post-processing or external context.

## Detailed design

Use a character sequence that would not otherwise appear in the [`path`](https://www.rfc-editor.org/rfc/rfc3986#section-3.3) component of an HTTP request to the bundler, namely `//&`, as an alternative delimiter in place of `?`, and percent-encode any `?` characters appearing within the subsequent query string, so that the result is a valid `path`,  encoding both the original path and query string data.

### Handling by bundlers
If the client sends a URL in this format, the bundler should respond as if `?` was seen in place of the alternative delimiter, with the following exception:
 - URLs of mixed format with `//&` preceding `?`, i.e. `.../path//&foo=bar?foo=baz&bar=quux` should be interpreted leniently, as if they were given fully in the alternative format, i.e. `.../path//&foo=bar&foo=baz&bar=quux`.

By construction (see "Choice of delimiter" below), there is no ambiguity introduced with standard URLs - the presence of the delimiter in the path is sufficient to infer that the input is encoded according to this RFC.

In addition, bundlers should *emit* URLs encoded in this form in any "source URLs" provided to the client, including in HMR payloads and `//# sourceURL` comments. The client will be expected to pass these encoded source URLs to the engine, where applicable, so that they are emitted in Error stacks, etc.

Bundlers *may* emit an encoded "source URL" in response to an HTTP bundle request via the `Content-Location` HTTP header, matching (after URL resolution) any `//# sourceURL` in the bundle output. By passing this encoded URL to the JavaScript engine (and in particular, JavaScriptCore), the client can generate usable stack traces without any change to the URL on which it *requests* the bundle.

## Drawbacks

 - The proposed URL format is unfamiliar, and standard libraries and tools won't be able to encode/decode build parameters into/from URLs in this format. An exposed API or package would help mitigate.
 - Maintenance/complexity of supporting an additional URL format.

## Alternatives

### Encode+decode within React Native runtime
Encode the source URL passed to JavaScriptCore to a format that won't be stripped, and need not be a URL (e.g., base64-encode the entire URL), catch errors within the runtime close to where they are thrown, and post-process stacks to decode and restore the original URLs before sending a `/symbolicate` request.

Pros:
 - No changes required to Metro, fix closely coupled to specific issue and may be removed if/when it becomes unnecessary.

Cons:
 - Relies on being able to catch and decode the stack before consumption - not a solution for all potential consumers of error stacks, (eg., errors caught within user code, uses of `new Error().stack`, remote debugging with Safari). 

### Choice of delimiter
Bundle requests begin with a representation of the project-root-relative path of an entry file on the file system, with directory separators represented as `/` (URL path separators). Clients are not required to percent-encode characters within the file path except as required by RFC3986. Because of this, and the almost unconstrained range of characters permitted in POSIX file paths, there are very few options for a delimiter that will not collide with potentially pre-existing bundle request URLs, and allow a non-breaking implementation of this RFC.

However, there are a small number of potentially useful constraints on POSIX file path segments:
 1. They may not contain a `null` character, so the corresponding percent-encoding `%00` will never appear in the path component of a bundle URL.
 2. They may not be empty strings, so consecutive path separators `//` should never appear.
 3. (If we further assume that the bundle URL only ever contains a "normalised" file path, the sequence `/./` should never appear.)

For future-proofing, we would like to choose a delimeter that allows encoding of the [`Fragment`](https://www.rfc-editor.org/rfc/rfc3986#section-3.5) (which is also stripped by JSC) in a consistent fashion, though doing so is beyond the scope of this RFC as fragments are not currently in use in bundle URLs.

`%00` may be used to terminate a POSIX path, but it could not be used to separate an encoded query string from an encoded fragment, due to potential ambiguity with an encoded `null` within a query string, with the fact that it is already percent-encoded.

`/./` makes the arguably fragile assumption of normalisation, and is more verbose than `//`.

`//` may be used to separate query from fragment if we required that this sequence *within* a query string must be percent-encoded. In order to distinguish a query delimiter from a fragment delimiter, so that either one or both may appear in a JSC-safe URL, we may add a character to the sequence, such as `//&` for queries and `//_` for fragments. The use of `//&` is chosen for readability and consistency with a query string of `&`-separated `key=value` pairs.

One notable caveat is that `//` may not be used at the very start of a path, due to ambiguity with the start of [`authority`](https://www.rfc-editor.org/rfc/rfc3986#section-3) (e.g. `//example.com`). Bundle URLs may not have empty paths, so handling this case is not in scope for this RFC, but potentially a combination of the above, such as a path beginning `%00//`, (a "null" path) could be used in this case.

## Adoption strategy

Metro and React Native patch releases will be required. No change is required from users.

## How we teach this

React Native users don't need to be aware of this explicitly. This RFC is intended to communicate the change with maintainers of CLIs and bundlers that may need to make changes to accommodate new URLs.
