# 2020-03-11 React Native Issue Management Meeting

We looked at the most recent issues that are tagged as "Needs: Triage": https://github.com/facebook/react-native/issues?q=is%3Aissue+is%3Aopen+label%3A%22Needs%3A+Triage+%3Amag%3A%22

We had a chance to go through 8 issues and walk through the triage process. We discussed how the bot manages issues and what aspects of the process are not yet automated. We'll cover the outcome of each issue's triage process for today's meeting notes; in future meeting notes, we might only call out issues that required escalation or otherwise didn't meet our basic issue triage process. We are doing this with the goal of surfacing opportunities to improve how we handle issues in the React Native repository.

## Issues:

- Fetching api response too slow when debugging mode is off: https://github.com/facebook/react-native/issues/28283 
    - This being an issue that depends on a network call to an API, we decided to ask for a minimal reproduction in order to eliminate the possibility this is caused by something outside of React Native itself. 
	- In cases like this, where the issue deals with slowness during the processing of a network call's response, we like to see a call being made to a known API endpoint (e.g. something anyone can reproduce themselves, without access to the original author's app server). We need the original author to spend more time digging into whether the slowness is present in the network call itself (e.g. for some reason, are they seeing that network calls within React Native take an order of magnitude longer to respond than when tested using a JS repl elsewhere? Or is the slowness happening as part of the json parsing of the response?) We need the original author to put in the work to ascertain this is an issue in React Native itself.
	- We asked for a Snack, even though the issue requires switching between development and release. The example code used to create a minimal repro for the Snack can be useful to whoever investigates the issue later.
- PropTypes to prop-types, Error evaluating React.PropTypes: https://github.com/facebook/react-native/issues/28282
    - The environment info for this issue shows React Native 0.55 being used, but 0.61 is the latest stable as of this writing on March 11, 2020. We ask that everyone try the latest version before filing an issue, in order to eliminate the possibility that the issue may have been solved already. 
	- Note that caution must be taken when looking at older issues. The author may have indeed verified the issue on what happened to be the latest release when they originally filed the issue. 
	- The react-native-bot will only comment on an issue at creation time if the version listed does not match latest in the releases tab on GitHub. The bot will re-check if the issue is edited, but only to remove the label. It will not remove its own comment if it finds the issue is now using the latest version.
- Metro bundler does not start: https://github.com/facebook/react-native/issues/28281
	- This was marked as a duplicate of a closed issue by someone in the thread.
	- We looked at the closed issue, and found it was closed after someone suggested they should post it to the CLI repo (https://github.com/react-native-community/cli/issues/484).
	- We let the author know that their issue was marked resolved there. If they need to, they can open a new issue in the cli repo.
- Android ignores unicode soft hyphen: https://github.com/facebook/react-native/issues/28279
	- This issue makes good use of images to display the problem.
	- We labeled it as affecting the Text component, as well as being an issue that described a disparity in how different platforms behave.
- Crash in RCTWebSocketModule: https://github.com/facebook/react-native/issues/28278
	- Crash reports are not a good fit for the repository. We ask the authors to investigate further and determine what the minimal repro is that causes the crash. Without this, it's unlikely anyone can help the author ascertain what causes the crash as we don't have access to the app's source code or the repro steps. 
	- Even if we did have the source code and repro steps, it's out of scope of the repo to investigate crashes in individual apps without a minimal repro that can be reproduced in a Snack or a brand new "rn init" app.
- StatusBar and Flexibility: https://github.com/facebook/react-native/issues/28276
	- Good use of <details> and images to demonstrate the issue. Repro steps describe how one might get to a repro, but we do need actual code (ideally a Snack). With a repository of this size, the bar is set so that the person reporting an issue is responsible for doing the work to provide the repro.
- Duplicate protocol definition of RCTBridgeModule is ignored: https://github.com/facebook/react-native/issues/28273
    - This issue mentions React Native 0.43.4 and Xcode 11.3. That version of React Native is very, very old as of this writing. Furthermore, Xcode 11.3 was released recently and it has only been tested with 0.61 and 0.62. Highly recommend they verify if this is still an issue on RN 0.61.
- CxxNativeModule: https://github.com/facebook/react-native/issues/28271
	- This issue does not make full use of the issue template, but it does mention having being tested on 0.62, which is still in release candidate phase as of this writing. We do want to encourage more folks to test the release candidate versions, therefore we can give this issue a pass for not adhering to the template.
	- Flagged as affecting the RC, and asked if the author might be interested in sending a pull request with a fix.
	
## Areas of opportunity:
	
The needs-response bot is now enabled in the repository. It will remove the "Needs: Author Feedback" label whenever the original author comments on an issue. It will not, however, add a label such as "Needs: Attention" after the author leaves a comment. Therefore, it's possible to lose track of issues where the author is waiting to hear back from us. We will leave these issues tagged as "Needs: Triage" for now. 

## Action Items
- [ ] @hramos: Investigate why the react-native-bot might fail to catch that an old version of RN is being used when the original author edits their issue and fills out the Environment info section. See Issue #28282.
- [ ] upforgrabs: Automate adding the attention label when the original author responds to an issue previously labeled as "Needs: Author Feedback"

## Attendees:
- Expo: @cruzach
- SumoLogic: @safaiyeh
- Microsoft: @chrisglein
- Facebook: @TheSavior, @hramos	
