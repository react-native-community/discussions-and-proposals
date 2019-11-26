# Deeper collaboration with Microsoft : November 21, 2019 meeting
A few key stakeholders from Facebook's React Native team and Microsoft teams met for a summit at Redmond a couple of weeks ago and discussed several topics over the course of 2 days. Our takeaways from the summit will make its way into a blogpost shortly. A lot of concrete opportunities to work more closely together were identified across the summit and we’re starting with a weekly meeting series between key stakeholders to begin to plan and make progress on those.

We kicked off the first of these meetings last week. Below are some of the high level topics discussed. We will ensure to post notes from our working meetings publicly in this repo as we work out how to broaden it and bring in more participants from the community as well. 

## Proposal for dealing with platform specific features
React Native lean core contains several features that work only on iOS/tvOS/Android etc., Similarly, as support for React Native for Windows and Mac have been bootstrapped, there are corresponding features that tend to be platfrom specific, but still meaningfully part of the lean core API set. We will be reviewing a set of options including coding practices, PR principles and documentation concerns for dealing with props that are inherently platform specific and cannot be implemented across all platforms as well as props that may originate in one (or) more platforms and upon broader vetting and validation becomes supported over time across platforms. 

## CI infrastrcuture 
A v-team has been identified to put together recommendations for a common and consistent CI infrastructure across facebook/react-native repo, microsoft/react-native-windows repo as well as react-native-community repos. Once some recommendations have been identified, we will share them with the community and expect to begin rollout post January 2020. In addition, a few folks will also be working on bootstrapping a benchmarking suite running on CI for every supported platform to help identify and get ahead of performance regressions, memory leaks, App size concerns etc., Static analysis tools and recommendations in this area will be published as we bootstrap.
Post these, we will be working on a broader test infrastructure and validation suite for react-native with consistent documentations and help for best practices and recommendations in this area. 

## Flow-Typescript bridging
Collaborations have been kicked off to help create any Codegen based automated bridging solution for supporting Typescript better as well as some processes to ensure quality of Typescript definitions as new APIs get added. 

## Instance management
The teams from Microsoft and Facebook shared each other's designs for better and high performance instance management in React Native. A lot of concrete ways to converge these designs for an overall better architecture were identified and we are excited to collaborate on improvements in this space.

## Community and ecosystem
In order to support an ecosystem of scale across more platforms, a working group will be publishing some processes, principles and guidelines for creating and maintaining high quality community modules with support for latest versions and all platforms including Windows and Mac. 

There is a lot more to come. Stay tuned!

