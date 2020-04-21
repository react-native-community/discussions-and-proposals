---
title: Adding a SectionList dataKeyExtractor prop
author:
- Rishabh
date: 22/04/2020
---

# RFC0000: Adding a SectionList dataKeyExtractor prop

## Summary

For now SectionList Usage doesn't allow user to specify the section's array data key and because of which user needs to process the data to make it look like this.
```jsx
const DATA = [
  {
    title: "Main dishes",
    data: ["Pizza", "Burger", "Risotto"]
  },
];


```
## Basic example

But if users data is somewhat like this.

```jsx
const DATA1 = [
  {
    title: "Main dishes",
    items: ["Pizza", "Burger", "Risotto"]
  },
  {
    title: "Main dishes",
    items: ["Pizza", "Burger", "Risotto"]
  },
];
```

A single prop can avoid the processing of data.

```jsx
<SectionList
  sections={DATA1}
  dataKeyExtractor={(index) => ("items")} 
  // if no value is returned/ prop missing then by default "data" will be considered as array key.
  // avoiding any breaking change
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => <Item title={item} />}
  renderSectionHeader={({ section: { title } }) => (
    <Text style={styles.header}>{title}</Text>
  )}
/>
```

## Motivation

This change will avoid redundant data processing everytime a user's data has changed or if is not of the format required by sectionList.

## Detailed design

The goal is to provide the new API and also keeping it backward compatible.
Changes in SectionList.js and VirtualizedSectionList.js will be needed.

## Drawbacks


## Alternatives


## Adoption strategy

As it won't be a breaking change it won't be painfull to adopt and will be helpful to avoid data processing.

## How we teach this

We will need to update the React Native documentation to introduce new Prop.

## Unresolved questions
i would like to discuss more on how i can contribute to this change and then if needed i can send a PR with required changes.
