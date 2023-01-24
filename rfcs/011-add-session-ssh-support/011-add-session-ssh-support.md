- Start Date: 2023-01-20
- Status: Proposed

# Add ssh support to sessions

## Summary

> One paragraph explanation of the change.

An important requirement which must be met by Renku is to enable users to
log in to their Renku session via SSH; a key driver for this is to support
working with VS Code within Renku sessions.

This has been discussed within the team and an imterim solution is currently
being developed; however, that solution is suboptimal as it requires creating
a jumphost which has access to the cluster and users need to log in via the
jumphost. This RFC details an alternative solution which is more in line with
the Renku design and incorporates operational concerns more explicitly.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

There is a
[Shape-up](https://www.notion.so/Support-RenkuLab-compute-access-from-local-terminal-SSH-VSCode-f896d3b391c94bcc87c56e375eb531d6)
document which provides the motivation for introducing this functionality.

## Problem Definition

> This section should include a detailed description of the problem and, importantly,
include any specific constraints that arise. It can be as technical as required.

The basic problem is to provide low friction, secure access to user sessions
via ssh. 

The standard ssh keypair workflow is as follows:
- create keypair
- ensure public key is stored on server to be accessed and ssh daemon is running
- log in to session for the first time
- confirm validity of remote server host key
- access remote session (if keypair matches)

From a user perspective, we need to take the following into account:
- user may want to use an existing keypair which is stored in `$HOME/.ssh`
- user may have existing keypairs but wants to have a new keypair for this scenario
- user may have a keypair which is accessible via the `ssh-agent`
- user may want to use PuTTY? (I've no idea how this intersects with VS Code)
- user should be able to log into the session with a simple `ssh` command
  - a standard `ssh <username>@<session>` should enable the user to log in
- user should obtain a sensible shell configuration when they log in
  - (not sure if something like devcontainer is required here)

***(It may be necessary to differentiate the above into mandatory and nice to
have).***

From a system perspective, the following needs to be taken into account:
- 

***(It may be necessary to differentiate the above into mandatory and nice to
have).***

We need to de

## Possible solutions


## Drawbacks

> Why should we *not* do this? Please consider the impact on users,
on the integration of this change with other existing and planned features etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Rationale and Alternatives

> Why is this design the best in the space of possible designs?

> What other designs have been considered and what is the rationale for not choosing them?

Two other designs have been considered:

> What is the impact of not doing this?

## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

> What parts of the design do you expect to resolve through the implementation of this feature before stabilisation?

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
