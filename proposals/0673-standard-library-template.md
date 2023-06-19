# RFC0673: Standard Library Template

| title | author | date |
| ----- | ------ | ---- |
| Standard Library Template | Burak Güner <md.burak.guner@gmail.com> | 2023-06-19 |

# Summary

We are planning to introduce a standardized template for React Native libraries.

# Motivation

Currently, there are numerous options available when creating a React Native library. Developers can choose between different languages (Java/Kotlin, Objc-Swift, C++), decide whether to include a view or not and determine whether to support the new architecture or not.

This situation leads to the following problems:

### Difficulty in upgrading the library to support newer versions of React Native

Having multiple options makes it challenging to upgrade the library to support newer versions of React Native. By focusing on a single template and utilizing the [library upgrade helper](https://github.com/react-native-community/rn-diff-lib-purge), upgrading libraries should become easier.

### Excessive code/control in user space

We are granting users too much control, which makes maintaining and supporting the library more difficult. For example, when using Gradle 8.0 on React Native 0.73, the Java 17 requirement introduces issues related to specifying the namespace. More details can be found in [this issue](https://github.com/callstack/react-native-builder-bob/issues/394).

### Difficulty in supporting multiple templates

Each new version of React Native must be compatible with the library template. This means extensive testing is necessary for each new React Native version, resulting in a significant workload. Supporting numerous templates becomes impractical.

### Challenges for new developers in starting library development

With numerous options available, it may be challenging for new developers to begin developing libraries. We can simplify this process by providing a standard template.

# Detailed design

## What is the golden template?

The golden template will be the recommended approach for developing React Native libraries. It should closely resemble the app template as much as possible.

### Languages

The golden template will initially support Java and Objective-C, with Kotlin support added later.

### Architecture

The template will support both the current and the new React Native architectures.

### Minimum Supported Versions

|  | Oldest Version | Newest Version |
| --- | --- | --- |
| Current Architecture | 0.64.0 | Latest |
| New Architecture | Latest version at the time we are creating the template | Latest |

# Drawbacks

- Maintaining the template and ensuring compatibility with the latest React Native version will require adding a CI pipeline to React Native's repository, increasing responsibilities for the core repository.
- Migrating existing libraries to the standard template may pose challenges.
- Developers can still choose to develop their libraries from scratch, but the emphasis on using the template may make it harder for those who prefer starting from scratch.
- Flexibility in choosing the language and architecture will be sacrificed.

# Alternatives

# Adoption strategy

We can utilize the [create-react-native-library](https://github.com/callstack/react-native-builder-bob) tool to initialize libraries based on the template and host the template itself. This tool is already used by library developers, making it convenient to adopt. We need to actively encourage library developers to utilize the template, which involves updating the documentation and adding a dedicated section for the template. Additionally, reaching out to library developers and requesting their adoption of the template can help promote its usage. Existing React Native apps will not be affected by this change.

# How do we teach this

We can add a new section to the official React Native documentation to explain the template. To simplify understanding and adoption, we can refer to this template as the "golden template."

# Unresolved questions

- What are the next steps to support out-of-tree platforms?
- How can we encourage library developers to adopt the template?
- How can we reduce the amount of code in user space?
