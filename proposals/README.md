# Planning and Proposals for React Native

Many changes, including bug fixes and documentation improvements, can be implemented and reviewed via the normal GitHub pull request workflow in the React Native project. Some changes are substantial enough to need additional planning, and others have enough impact to need additional community input. We ask that both of these use the following proposal format below when practical.

[Active Proposal List](https://github.com/react-native-community/discussions-and-proposals/pulls?q=is%3Aopen)

## The process

In short, to make a major change to React Native, make a proposal and submit it as a pull request. The community will review it and may opt to approve it. At that point, the proposal is accepted and may be implemented.

- Fork this repository [react-native-community/discussions-and-proposals](http://github.com/react-native-community/discussions-and-proposals)
- Copy `proposals/0000-template.md` to `proposals/0000-my-feature.md` (where 'my-feature' is the title in kebab case; don't assign a number yet).
- Fill in the proposal. Put care into the details: **Proposals that do not present convincing motivation, demonstrate understanding of the impact of the design, or are disingenuous about the drawbacks or alternatives tend to be poorly-received**.
- Submit a pull request. As a pull request, the proposal will receive feedback from the larger community, and the author should be prepared to revise it in response.
- Build consensus and integrate feedback. Proposals that have broad support are much more likely to make progress than those that don't receive any comments.
- Eventually, the team will decide whether the proposal is a candidate for adoption.
- A proposal can be modified based upon feedback from the team and community. Significant modifications may trigger a new final comment period.
- A proposal may be rejected by the team after public discussion has settled and comments have been made summarizing the rationale for rejection. A member of the team should then close the associated pull request.
- A proposal may be accepted. A team member will merge the proposal's associated pull request, at which point the proposal will become adopted.

## After acceptance

Once a proposal is accepted, then authors may implement it. This may mean submitting a pull request to the React Native repository or putting some other process into place. Acceptance however does not mean that resources are committed to the work; instead it means that the group is open to the change taking place.

Modifications to accepted proposals can be done in followup PRs.

## Implementing a proposal

The author of a proposal is not obligated to implement it. Of course, the proposal author (like any other community member) is welcome to post an implementation for review.

## Reviewing a proposal

Periodically, the team will attempt to review the active proposals. We try to accept proposals at the monthly team meeting, and actions are recorded in the meeting minutes. Every accepted feature should have a core team champion, who will represent the feature and its progress.

**React Native's proposal process owes its inspiration to the [Rust RFC process], the [Ember RFC process], the [Yarn RFC process], and the [React RFC process]**

[yarn rfc process]: https://github.com/yarnpkg/rfcs
[react rfc process]: https://github.com/reactjs/rfcs
[rust rfc process]: https://github.com/rust-lang/rfcs
[ember rfc process]: https://github.com/emberjs/rfcs
