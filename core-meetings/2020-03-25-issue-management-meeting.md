# 2020-03-25 React Native Issue Management Meeting

## Agenda:
- Issue management meeting goals
- Workflow enhancements
- Go through Needs: Triage issues
- Go through Needs: Attention issues
- Discuss areas of improvement
- Create action items

## Issue management meeting goals

Héctor shared his thoughts on issue management:
- Issue triage should be a low-friction and handled by a large group of community members
- Issue management meeting is where we surface improvements to the issue triage process and handle escalation paths from community triage
- Folks in the weekly meeting might not necessarily do the triage themselves, instead they
    - Shape how issues are managed
    - Define what a "good issue" is, the final state we want every issue to get to, where they are ready to be looked at by someone who is interested in providing a fix (which could be one of a RN FB team member, RN core team member, or a community member who has expressed interest)

Getting there:
- For the next week:
    - Meeting members to go through issues 3x/week and triage. Intent is to dogfood the process and surface workflow improvements.
    - Test out the new automations and surface any additional labels or canned responses that would be helpful.
    - Flag any issues that were tricky to triage.
- Onboard community members into the issue triage group:
    - Let Eli or Héctor know, and we will grant them issue management permissions. This allows them to apply labels, close, reopen issues, on the core RN repository.
    - React Native Wiki contains issue triage workflow to onboard new folks: https://github.com/facebook/react-native/wiki/Issues#triaging-issues
- After dogfooding workflow,
    - Create an escalation path for issue triage group: applying a "Needs: Issue Manager Attention" label.
    - Meeting to switch focus to issues tagged as such.

## Workflow enhancements

The [label-actions](https://github.com/dessant/label-actions) GitHub App has been enabled for the core repository. It provides the following functionality:

- Automatically adds a comment on issues whenever any of these labels is applied:
    - "Needs: Verify on Latest Version"
    - "Needs: Environment Info"
    - "Needs: Issue Template"
- ...and also closes the issue when these labels are applied:
    - "Type: Invalid"
    - "Type: Question"
    - "Type: Docs"
    - "Resolution: For Stack Overflow"

The full configuration file can be found in [`.github/label-actions.yml`
](https://github.com/facebook/react-native/blob/master/.github/label-actions.yml). Please feel free to send a pull request with enhancements to this config.

The [needs-attention](https://github.com/hramos/needs-attention) GitHub Action has been enabled, and will trigger whenever a new comment is added to an issue. The goal is to manage the "Needs: Author Feedback" label and make sure we do not lose track of issues where we asked for more information. If an issue with this label gets a new comment from the original author of the issue, this action will replace the "Needs: Author Feedback" label with a "Needs: Attention" label. This is the same process used in the `react-native-windows` repository. 

The workflow is configured in [`.github/workflows/needs-attention.yml`](https://github.com/facebook/react-native/blob/master/.github/workflows/needs-attention.yml), and the action itself is hosted at https://github.com/hramos/needs-attention. Pull requests that enhance this workflow are welcome.

## Issue triage

We went through a small number of issues with the goal of demonstrating the new label actions in use. We looked at [Issues that Need Triage](https://github.com/facebook/react-native/issues?q=is%3Aissue+is%3Aopen+label%3A%22Needs%3A+Triage+%3Amag%3A%22), as well as [Issues that Need Attention](https://github.com/facebook/react-native/issues?q=is%3Aissue+is%3Aopen+label%3A%22Needs%3A+Attention%22+).

## Action Items

- See "Getting there" above. Everyone to go through issues ~3x/week before our next meeting, a single page of GitHub search results should be enough (25 issues). Remove the "Needs: Triage" label for each issue you've triaged, to avoid double dipping.
- Identify folks in your community who would be interested in triaging issues.
- Héctor to update Wiki with latest workflow decisions: https://github.com/facebook/react-native/wiki/Issues

## Attendees:
- Coinbase: @imownbey
- Expo: @cruzach
- SumoLogic: @safaiyeh
- Facebook: @TheSavior, @hramos	
