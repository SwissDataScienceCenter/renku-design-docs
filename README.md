# Renku Design Documentation - RFC

New features in Renku often require complex coordination and planning among
various components. They also require a careful study and understanding of
technology options and choices. Some features might also have far reaching
consequences that might affect users or various components in a major,
interdependent way.

For such large undertakings, the RFC (Request For Comments) process provides a
persistent and controlled path for new features and changes to enter the
codebase.

This repository hosts the design documentation for Renku features. By following
an open, PR-review process, it also gives users and other third parties a chance
to see and contribute to the design of these features.

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

## How to submit an RFC

- Copy the `rfc-template.md` file and name it like `0000-my-feature.md` (with an
  incrementing number and appropriate title). Make a subdirectory in the
  `design/` directory for your RFC.

- Fill in the copied template with design details, taking care to address the
  motivations, impact and design criteria/limitations

- Submit a pull request for the RFC - open a discussion on discourse to let the
  Renku community know and solicit comments

- After the pull request is approved and merged, the RFC should be considered
  for entering into the development process.

- A merged RFC can be modified further through additional PRs.

Once an RFC has been merged, it can be broken up into issues and implemented
as usual. If the work is spread across multiple repositories, the issues should
be organized into an epic in the main `renku` repository.

RFCs are not meant to be set in stone and can be subject to change. As with any
large undertaking, assumptions can turn out to be false, better solutions may
present themselves or they may turn out to not be feasible after all. Therefore
RFCs can change after being accepted, but care should be taken to make them as
robust as possible.
