# Renku Design Documentation

New features in Renku often require complex coordination and planning among
various components. They also require a careful study and understanding of
technology options and choices. Some features might also have far reaching
consequences that might affect users or various components in a major,
interdependent way.

## Organization

This repository differentiates between two types of design documents:

### Feature pitches

For better visibility of feature design and implementation, each feature is
fully described on a high level in a feature pitch document, found in the
`feature pitches` directory. Feature pitches avoid detailed specification on
purpose; the implementation details are to be figured out by the small,
dedicated build team when the feature is accepted for implementation.

The level of description of feature pitches should also allow interested
external parties (e.g. end-users) to participate in the feature design and
vetting process.

### Request-for-comment (RFC)

For large architectural or refactoring undertakings, the RFC (Request For
Comments) process provides a persistent and controlled path for changes to enter
the codebase. RFCs are much more detailed than feature pitches and should
provide enough information such that issues can be made directly from the
information found in the RFC.

## When to submit a feature pitch document

Any time a user-facing feature is to be developed, it should be accompanied by a
feature pitch document. For guidance on how to structure this document and
"shape" the feature before implementation, see [a description of
shaping](https://basecamp.com/shapeup/1.1-chapter-02).

## When to submit an RFC

An RFC should be submitted whenever a substantial change is being considered for
implementation. What constitues a substantial change is not set in stone, but
may include:

- Adding new service endpoints/changing service endpoints in way that break
  downstream code

- Adding new groups of CLI commands, adding new CLI commands that have a big
  impact on user interaction with renku

- Significant changes to the json-ld metadata that go beyond e.g. adding a
  simple field

- Adding a new service that enables new ways of managing datasets

## How to submit a feature pitch or an RFC

- Copy the appropriate template (`feature-pitch-template.md` or
  `rfc-template.md`) and name it like `000-my-feature.md` (with an incrementing
  number and appropriate title). Make a subdirectory in the `feature pitches` or
  `rfcs/` directory for your document.

- Fill in the copied template with design details, taking care to address the
  context, motivations, impact and design criteria/limitations

- Submit a pull request - depending on the feature or RFC, open a discussion on
  discourse to let the Renku community know or solicit comments from specific
  interested parties.

- After the pull request is approved and merged, the feature pitch or RFC should
  be considered for entering into the development process.

Feature pitches should be crafted in a way that allows room for interpretation at
the implementation stage. As such, they should not need to be modified after they
have been vetted by team leads and other stakeholders.

On the other hand, since RFCs are more detailed and technical, they are expected
to evolve as implementation proceeds. RFCs are not meant to be set in stone and
can be subject to change. As with any large undertaking, assumptions can turn
out to be false, better solutions may present themselves or they may turn out to
not be feasible after all. Therefore RFCs can change after being accepted, but
care should be taken to make them as robust as possible. After an RFC is
accepted and merged, issues should be made to track progress of the
implementation.
