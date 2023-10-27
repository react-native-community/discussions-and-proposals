---
title: Golden Template for create-react-native-library
authors:
- Burak Güner (@atlj)
- Satyajit Sahoo (@satya164)
- Lorenzo Sciandra (@kelset)
- Nicola Corti (@cortinico)
- Pavlos Vinieratos (@pvinis)
date: 2023-10-20
---

# RFC0721: Golden Template for create-react-native-library

## **Summary**

This RFC aims to document the currently implicit React Native library API and provides a default implementation for it. The RFC also addresses the current challenges caused by the absence of an official library template for React Native, and outlines technical specifications for the proposed template.

The template aims to utilize the existing community tool [create-react-native-library](https://github.com/callstack/react-native-builder-bob) and proposes necessary adjustments to the templates and UX to match the specifications outlined in the RFC.

## **Rationale**

Currently, there is no official way of creating libraries for React Native. [The React Native documentation](https://reactnative.dev/docs/native-modules-setup) recommends [create-react-native-library](https://github.com/callstack/react-native-builder-bob) as a community tool and gives a brief introduction to it. The lack of an official template leads to the following issues:

- It’s hard to communicate with all the community library authors when there is going to be a breaking change for them with newer versions of React Native.
    - One of the most obvious examples is the rollout of the New Architecture. The new architecture introduced numerous changes to library structure and has been tricky to follow, as they also vary between React Native versions.
    - Another recent example is the changes required by the adaptation of Android Gradle Plugin 8.0. All React Native libraries targeting Android needed to update their libraries and, as there was no standard way to do it, most of the libraries went different ways. You can find the full issue [here](https://github.com/react-native-community/discussions-and-proposals/issues/671).
    - The recent outage experienced by [React Native MMKV](https://github.com/mrousavy/react-native-mmkv), caused by a shift in dependencies within [fbjni](https://github.com/facebookincubator/fbjni), highlights the critical need for an official library template in the React Native ecosystem; this incident emphasizes how the absence of standardized guidelines can lead to compatibility issues, antipatterns, and unintended consequences in library development. You can find the full issue [here](https://github.com/facebookincubator/fbjni/issues/85).
- It’s not beginner friendly since they can be easily overwhelmed by the many options they have while creating React Native libraries.
- The lack of guidance on how a library should interact with the code in React Native core has led maintainers to find their own ways, sometimes hacky or unexpected for the RN core team, which in turn led to fragmentation and unwanted breakages for libraries.
- There is no guarantee that generated libraries will work with newer versions of React Native as the changes inside React Native aren’t tested against the library templates.

While we can keep things in their current state, having an official library template would yield the following benefits:

- A library counterpart of [React Native app upgrade helper](https://react-native-community.github.io/upgrade-helper/) can be created for upgrading the libraries that follow the official template. This would make it easier to support newer versions of React Native for libraries.
- In order to ensure newer versions of React Native are compatible with the library template, additional CI pipelines can be added to the core React Native repository. They would test every change on React Native against the library template.
- The required knowledge level would be less for creating native libraries since new developers could only focus on officially supported languages and interfaces instead of getting overwhelmed with the number of options they have.
- The `create-react-native-library` maintainers could prioritize maintaining the standard template. This would in turn make it more stable.

## **The official library template specifications**

To maintain consistency within the React Native ecosystem, the technical decisions for the official template align with the [official React Native app template](https://github.com/facebook/react-native/tree/main/packages/react-native/template).

### Programming languages

The official template will support **Java**, **Objc**, and **TypeScript** initially, as these are the languages used in the official React Native app template. However, there are plans to incorporate **Kotlin** to replace **Java** with the release of **React Native 0.73**.

### **Architecture support**

The official template should support **both the new and the current architectures** of React Native. This means it should be **backward compatible**. To see more about backward compatibility on React Native, please visit [https://reactnative.dev/docs/next/the-new-architecture/backward-compatibility](https://reactnative.dev/docs/next/the-new-architecture/backward-compatibility). It’s also important to note that the new architecture support is only expected to work with the latest version of React Native and not with prior versions.

### **What will be inside the official library template?**

The official library template will consist of four main folders:

- `ios`: Native iOS source code that includes a Native Module and a Native View.
- `android`: Native Android source code that includes a Native Module and a Native View.
- `src`: Javascript source code.
- `example`: An example app to test the library, created by running `npx react-native init` and making necessary changes for correct library linking.

[React Native Builder Bob](https://github.com/callstack/react-native-builder-bob) will come pre-installed with the template for **building**, **generating the changelog**, and **publishing** the library.

Apart from the necessary library code, the template will also include a *GitHub Actions* workflow to **lint**, **type-check**, and **build** the library. The workflow will also **build the example app** for Android and iOS.

### **How should `create-react-native-library` look like**

Running `npx create-react-native-library` will invoke the CLI, which will prompt the user for inputs to customize their library. Two relevant questions out of eight are:

1. What type of library do you want to develop?
    1. The user will choose from several options, and the official template's settings will be marked as recommended.
    
    ![create-react-native-library's library type selection prompt](./assets/0721-crnl-library-type-selection.png)
    
2. Which languages do you want to use?
    1. Users will select their preferred programming languages, with Java and Objective-C recommended since the official template uses these languages.
    
    ![create-react-native-library's language selection prompt](./assets/0721-crnl-language-selection.png)
    

After the prompts, an example app will be generated using `npx react-native@latest init`. The resulting library will contain both a native module and a native view.

> In the current setup of **`create-react-native-library`**, we have distinct templates for native modules and native views. To streamline this and provide a unified solution, we can introduce a single template that covers both native views and native modules. This should also reduce the maintenance cost of the template.
> 

### **Library upgrade helper**

One of the primary goals of this proposal is to make it easier to upgrade libraries. Specifically to make sure libraries are compatible with newer versions of React Native.

Similar to the existing [React Native Upgrade Helper](https://react-native-community.github.io/upgrade-helper/), the library upgrade helper will initialize a new library every time there are updates inside `create-react-native-library`. It will then use diffs to compare versions, facilitating the upgrade process. Here is the detailed process:

1. **Initialization of a New Library for Comparison**: Whenever there is a new release inside `create-react-native-library`, the [GitHub Actions REST API](https://docs.github.com/en/rest/actions?apiVersion=2022-11-28) will trigger the building process for the library upgrade helper. The library upgrade helper will initiate the creation of a new library using the official template.
2. **Comparing Differences with Diffs**: The library upgrade helper will then compare the previous version of the library with the one created using the updated template. It will use diffs to identify the changes that need to be made to the existing library. The folder that contains the example app won’t be included in the diff and the user will be pointed to the app upgrade helper instead.
3. **Facilitating the Upgrade Process**: Once the differences are identified, the library upgrade helper will guide the library developers through the upgrade process. It will provide clear instructions on what changes are needed, what new features or APIs are available in the latest React Native version, and how to adapt the library to ensure compatibility.
4. **Documentation and Resources**: To support developers further, the library upgrade helper will link `create-react-native-library`'s changelog.

The primary purpose of the library upgrade helper is to reduce the burden on library developers during the upgrade process. Automating some of the tasks and providing clear guidance, streamlines the process and minimizes potential compatibility issues. This way, library developers can focus on delivering new features and improvements to their libraries while staying in sync with the latest advancements in React Native.

### **New CI pipelines on the React Native repository**

The proposal suggests adding a CI pipeline to the [core React Native repository](https://github.com/facebook/react-native) to test compatibility with `create-react-native-library`. This ensures that every change in React Native is tested against the library template.

Practically we’ll be adding four new CI jobs on React Native CircleCI’s setup as follows:

- `test_android_template_withlibrary_OldArc` - This will test the Android New App template against the Golden Template from CRNL with the Old Architecture
- `test_android_template_withlibrary_NewArc` - This will test the Android New App template against the Golden Template from CRNL with the New Architecture
- `test_ios_template_withlibrary_OldArc` - This will test the iOS New App template against the Golden Template from CRNL with the Old Architecture
- `test_ios_template_withlibrary_OldArc` - This will test the iOS New App template against the Golden Template from CRNL with the New Architecture

Those jobs will ensure that the new app template is continuously tested against the golden template.

### **Documenting the supported React Native versions**

Currently, there is no such section inside the [create-react-native-library repository](https://github.com/callstack/react-native-builder-bob)’s documentation to list the compatible React Native versions. To address this issue, supported versions can be tested manually.

### Versioning `create-react-native-library`

`create-react-native-library` will follow a versioning schema based on **`major.minor.patch`** as outlined below:

1. The **patch** version will be incremented for bug fixes.
2. The **minor** version will be incremented for updates to dependencies and the introduction of new features in the template. **Minor** versions will not remove support for any React Native versions.
3. The **major** version will be incremented for breaking changes, such as dropping support for specific React Native versions.

### **Minimum supported versions of iOS and Android**

The library template aims to support the same minimum versions of host operating systems as the app template. This translates to `13.4` on iOS and `6.0` on Android.

---

> The following points are possible additions to the proposal. They can be added in the future.
> 

### Using the [React Native Test App](https://github.com/microsoft/react-native-test-app) as the example app (possible addition)

> “React Native Test App (RNTA) provides test apps for all platforms as a package. It handles the native bits for you so you can focus on what's important: your product.” [-React Native Test App’s readme](https://github.com/microsoft/react-native-test-app/blob/trunk/README.md)
> 

React Native Test App can be used to test the libraries. It’s easier to maintain than an app created with the app template since it handles the native parts.

### Reducing the amount of boilerplate code in the template (possible addition)

As many libraries share some common logic, the proposal suggests moving some of the boilerplate code to the core template, simplifying individual library templates. You can take the [build.gradle file of a native module](https://github.com/atlj/official-template-proposal-module/blob/main/android/build.gradle) as an example. There are a lot of lines that are shared between many libraries.

---

## **Drawbacks**

The primary drawback of this proposal is the additional maintenance cost for the core React Native repository. The React Native team needs to actively monitor if their changes break the template, fix any regressions, and communicate the necessary changes to the template.

## **Alternatives**

Various alternatives to `create-react-native-library` exist, listed here in no particular order:

- [https://github.com/brodybits/react-native-module-init (not maintained)](https://github.com/brodybits/react-native-module-init)
- [https://github.com/brodybits/create-react-native-module (not maintained)](https://github.com/brodybits/create-react-native-module)

> Note: While using a template streamlines the library creation process, developers can still choose to start from a blank repository and customize it as needed.
> 

## **How to adopt it?**

Adopting the proposed official template and library upgrade helper involves the following steps:

1. **Updating `create-react-native-library`:** The first step is to update the **`create-react-native-library`** tool to make the official template the recommended option.
2. **Communicating with library owners:** To encourage existing React Native libraries to adopt the official template, proactive communication is essential. Library owners can be informed about the advantages of the official template, such as easier upgrades and better compatibility with future React Native versions. The communication might also include step-by-step guides on migrating existing libraries to the official template and utilizing the library upgrade helper for seamless upgrades. Further details won’t be covered here.
3. **Curating example libraries**: To promote adoption, we can compile a list of outstanding libraries that we intend to target for adopting the official template. These libraries will also serve as practical examples of best practices in React Native development, demonstrating the template's effectiveness across various use cases. The `create-react-native-library team` can proactively engage with these libraries to ensure compatibility. An early example of such a library could be [React Native Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler/).

## **Documentation update is needed**

Updating the React Native documentation is crucial to ensure that developers have comprehensive and up-to-date information about the official library template and the library upgrade helper. The following documentation updates are required:

1. **Official Library Template:** The documentation should include detailed instructions on how to use the **`create-react-native-library`** CLI tool with the official library template. Additionally, developers should be provided with examples and best practices to help them get started quickly.
2. **Library Upgrade Helper:** A dedicated section can be added to the documentation to explain the library upgrade helper's functionality. It should describe how the helper compares different library versions, provides upgrade instructions, and assists in ensuring compatibility with newer React Native releases. The documentation might offer step-by-step guides on how to utilize the helper effectively.
3. **Benefits and Advantages:** The documentation should clearly outline the benefits of adopting the official template and using the library upgrade helper. This section should emphasize the importance of keeping libraries up-to-date with the latest React Native versions, the improved stability resulting from using the official template, and how the helper streamlines the upgrade process.

## **Unresolved questions**

- How to encourage existing React Native libraries to use the official template?
- How should we tackle the support for out-of-tree platforms such as [React Native Windows](https://github.com/microsoft/react-native-windows)?
- How should we deal with JNI/JSI/C++ only user cases?
