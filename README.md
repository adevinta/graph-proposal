# Proposing Changes to Security Graph

This proposal process is based on the [Go proposal process](https://github.com/golang/proposal)
and has been simplified to fit our current needs.

## Introduction

The Security Graph project's development process is design-driven. Significant
changes must be first discussed, and sometimes formally documented, before they
can be implemented.

This document describes the process for proposing, documenting, and
implementing changes to the Security Graph project.

## The Proposal Process

The proposal process is the process for reviewing a proposal and reaching a
decision about whether to accept or decline the proposal.

1. The proposal author creates a brief GitHub issue in this repository
   describing the proposal.

   Note: There is no need for a design doc at this point.

2. A discussion on the issue aims to triage the proposal into one of three
   outcomes:
     - Accept proposal, or
     - decline proposal, or
     - ask for a design doc.

   If the proposal is accepted or declined, the process is done. Otherwise the
   discussion is expected to identify concerns that should be addressed in a
   more detailed design.

3. The proposal author writes a [design doc](#design-documents) to work out
   details of the proposed design and address the concerns raised in the
   initial discussion.

4. Once comments and revisions on the design doc wind down, there is a final
   discussion on the issue, to reach one of two outcomes:
     - Accept proposal, or
     - decline proposal.

After the proposal is accepted or declined (whether after step 2 or step 4),
implementation work proceeds in the same way as any other contribution.

## Detail

### Goals

- Make sure that proposals get a proper, fair, timely, recorded evaluation with
  a clear answer.
- Make past proposals easy to find, to avoid duplicated effort.
- If a design doc is needed, make sure contributors know how to write a good
  one.

### Definitions

- A **proposal** is a suggestion filed as a GitHub issue in this repository.
- A **design doc** is the expanded form of a proposal, written when the
  proposal needs more careful explanation and consideration.

### Scope

The proposal process should be used for any notable change or addition to the
Security Graph project.

Since proposals begin (and will often end) with the filing of an issue, even
small changes can go through the proposal process if appropriate. Deciding what
is appropriate is matter of judgment we will refine through experience. If in
doubt, file a proposal.

### Design Documents

As noted above, some (but not all) proposals need to be elaborated in a design
doc.

- The design doc should be checked in to [the proposal repository](https://github.com/adevinta/graph-proposal/)
  as `design/NNNN-shortname.md`, where `NNNN` is the GitHub issue number and
  `shortname` is a short name (a few dash-separated words at most). Clone this
  repository and submit a PR following the usual GitHub workflow.
- The design doc should follow [the template](design/TEMPLATE.md).
- The design doc should address any specific concerns raised during the initial
  discussion.
- For ease of review with GitHub, design documents should be wrapped around the
  80 column mark.
- Comments on GitHub PRs should be restricted to grammar, spelling, or
  procedural errors related to the preparation of the proposal itself. All
  other comments should be addressed to the related GitHub issue.

## Contributing

This project is in an early stage, we are not accepting external contributions
yet.
