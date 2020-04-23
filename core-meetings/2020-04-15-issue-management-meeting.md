# 2020-04-15 React Native Issue Management Meeting

## Agenda:
- Discuss common patterns observed in offline triage
- Go through Needs: Issue Manager Attention issues
- Go through Needs: Triage issues
- Go through Needs: Attention issues
- Discuss areas of improvement
- Create action items

## Notes

- No issues flagged [Needs: Issue Manager Attention](https://github.com/facebook/react-native/labels/Needs%3A%20Issue%20Manager%20Attention) this week
- Lots of issue traffic from upgrade issues to 0.62
  - [Type: Upgrade Issue](https://github.com/facebook/react-native/labels/Type%3A%20Upgrade%20Issue) will redirect to the [Upgrade Support repository](https://github.com/react-native-community/upgrade-support) and close it.
  - Action item: Add some communication in the bot tag response that they should come back if they are unable to resolve their issue due to an actual bug.
  - Potentially retire these: [DX: Upgrading](https://github.com/facebook/react-native/labels/DX%3A%20Upgrading), [Issue When Upgrading](https://github.com/facebook/react-native/labels/Issue%20When%20Upgrading).
- COVID cadence - offline every other week.
- [Needs: Verify on Latest Version](https://github.com/facebook/react-native/labels/Needs%3A%20Verify%20on%20Latest%20Version) - what's the support window? Do we tag issues on 61 now that 62 is out?
  - If they filed against the latest version when they opened the issue, that's fine (at least for N-1).
- Do we clear "Needs: Triage" if issue looks completely good?
  - Facebook team would still like to look through these for prioritization
  - Having a label that says "this issue is good" doesn't feel quite right
  - Suggestion: Add "positive tags" such as "has good repro", "good description", etc.
  - Of note, [Issue: Author Provided Repro](https://github.com/facebook/react-native/labels/Issue%3A%20Author%20Provided%20Repro) already exists, and we should use.
- Noticed that the bot is automatically applying Component/API labels
  - Sounds like was preexisting functionality, there was a bug, and it was fixed. Yay!
- Lot of Flipper issues?
  - Somewhat expected as it's new to 62. Now has [a label](https://github.com/facebook/react-native/labels/Flipper).
- Meeting cadence in light of COVID-19 challenges and reduced meeting bandwidth
  - Chris is going to send out a fresh invite (moving from Harini), moving to biweekly
  - Can do an offline checkin in off weeks, as we're trying to beef up offline triage anyway
- 0.62 issues and Expo
  - Expo won't be upgraded until ~June/July so snacks are not good for reproing 0.62 issues.
  - This is pretty much always a problem with updates to latest version (part of why snacks aren't a requirement)
  - Potential concern: Will Expo fall behind with the many-version updates expected on path to TurboModules?
- Tip: issues with the documentation should be labeled [Type: Docs](https://github.com/facebook/react-native/labels/Type%3A%20Docs) (will close issue)
- After meeting (and small offline follow ups) triage needed is at 127

## Attendees

- Microsoft: Chris Glein (@chrisglein)
- Coinbase: Ian Ownbey (@imownbey)
- Expo: Charlie Cruzan (@cruzach)
- SumoLogic: Jason Safaiyeh (@safaiyeh)
- Facebook: Héctor Ramos (@hramos)
