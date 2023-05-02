---
title: JavaScriptCore-safe bundle URLs
authors:
- Rob Hogan (@robhogan)
- Moti Zilberman (@motiz88)
date: 2023-05-02
---

# RFC0642: JavaScriptCore-safe bundle URLs

## Summary

Add support for a query-string-free URL format to bundlers, so that symbolication (etc.) is not broken by JavaScriptCore error stack manipulation.

## Basic example

`http://localhost:8080/project/index.bundle;&platform=ios`

Should be supported by Metro (and other bundlers) equivalently and in addition to the current query-string form:

`http://localhost:8080/project/index.bundle?platform=ios`

## Motivation

Recent versions of JavaScriptCore strip query strings from source URLs in stack traces (`Error.prototype.stack`).

To implement features such as Metro's `/symbolicate` endpoint correctly, we must be able to derive (a close approximation of) the source that was executed the client. To rebuild that source correctly we need to use the same build parameters. 

Ideally, for maximal compatibility with debuggers (etc.) that may intercept the raw error stack, the source URLs in the error stack should be sufficient, and should be valid URLs, without any post-processing or external context.

## Detailed design

Use a character sequence from the "reserved" set specified in RFC 3986, namely `;&`, as an alternative delimiter in place of `?`, and URL-encode any `?` characters appearing within the subsequent query string so that the result is a valid [`path`](https://www.rfc-editor.org/rfc/rfc3986#section-3.3).

If the client sends a URL in this format, Metro should respond as if `?` was seen in place of the alternative delimiter, with the following exceptions:
 - Wherever the bundle itself encodes its own URL, source map URL, or the URLs of other bundles on the same server (such as for lazy bundling), those should use the alternative format for any included query strings *if and only if* the original bundle request used the alternative format. Otherwise, standard query strings starting with `?` should be emitted.
 - URLs of mixed format with `;&` preceding ?, i.e. `.../path;&foo=bar?foo=baz&bar=quux` should be interpreted leniently, as if they were given fully in the alternative format, i.e. `.../path;&foo=bar&foo=baz&bar=quux`.

Because the alternative delimiter is made of reserved characters, it does not create ambiguity with standard URLs and may be escaped by percent-encoding for use as a param value, path component, etc. In particular, this supports the edge case of a valid POSIX file path containing the substring `;&` being the subject of a bundle request.

## Drawbacks

 - The proposed URL format is unfamiliar, and standard libraries/tools won't be able to encode/decord build parameters into/from paths in this format. An exposed API or package would help mitigate.
 - Maintenance/complexity of supporting an additional URL format.
 - Forking of bundle URL generation in React Native by platform/engine, assuming we target this URLs as narrowly as possible.

## Alternatives

## Encode+decode within React Native runtime
Encode the source URL passed to JavaScriptCore to a format that won't be stripped, and need not be a URL (eg, base64-encode the entire URL), catch errors within the runtime close to where they are thrown, and post-process stacks to decode and restore the original URLs before sending a `/symbolicate` request.

Pros:
 - No changes required to Metro, fix closely coupled to specific issue and may be removed if/when it becomes unnecessary.

Cons:
 - Relies on being able to catch and decoding the stack - not a solution for all potential consumers of error stacks, (eg., remote debugging with Safari). 

## Adoption strategy

Metro and React Native patch releases will be required. No change is required from the users.

## How we teach this

React Native users don't need to be aware of this explicitly. This RFC is intended to communicate the change with maintainers of CLIs and bundlers that may need to make changes to accommodate new URLs.

## Unresolved questions

 - How do we expose functions for encoding/decoding? Is a `metro` export sufficient? Should we publish a new package? Or document a pseudo-code algorithm?
