- Start Date: 2023-01-20
- Status: Proposed

# Add ssh support to sessions

## Summary

> One paragraph explanation of the change.

An important requirement which must be met by Renku is to enable users to
log in to their Renku session via SSH; a key driver for this is to support
working with VS Code within Renku sessions.

This has been discussed within the team and work is ongoing on an interim
solution which meets the basic requirements. This RFC provides the context,
supports discussion of possible solutions and will be used to document the
final agreed solution.

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

The standard ssh (keypair-based) workflow is as follows:
- create keypair
- ensure public key is stored on server to be accessed and ssh daemon is running
- log in to session for the first time
- confirm validity of remote server host key
- access remote session (if keypair matches)

From a user perspective, the following needs to be taken into account:
- user may want to use an existing keypair which is stored in `$HOME/.ssh`
- user may have existing keypairs but wants to have a new keypair for this scenario
- user may have a keypair which is accessible via the `ssh-agent`
- user may want to use PuTTY? (I've no idea how this intersects with VS Code)
- user should be able to log into the session with a simple `ssh` command
  - a standard `ssh <username>@<session>` should enable the user to log in
- user should obtain a sensible shell configuration when they log in
  - (not sure if something like devcontainer is required here)

From a system perspective, the following needs to be taken into account:
- Unlike HTTP/S, SSH does not have support for server identification and hence
  it is not trivial to perform aggregation of SSH sessions; further, this
  carries non-negligible security risks, so it is not a standard approach.
- The Renku platform should never see user private keys
- A solution which does not have many moving parts is definitely preferred
  (e.g. we should avoid publishing DNS entries dynamically if possible)
- Solutions which consume large amounts of IP addresses should be avoided as
  they can be costly
- A solution which supports monitoring in a convenient manner is desirable

***(It may be necessary to differentiate the above into mandatory and nice to
have).***

## Key Assumptions

The following key assumptions apply:
- a solution is required which can be deployed both in the cloud provider
  context and in the Switch/Openstack context
- we are not targetting the most basic users; we assume they understand the
  need for some security and are willing to put the time/energy required into a
  reasonably sensible ssh configuration
- password authentication to ssh sessions exposed to the Internet is not
  permitted

## Possible solutions

As this work has some kind of history, three solutions have been discussed in
various conversations - these are:
- an approach in which an ssh port is exposed on each session directly to the
Internet;
- an approach in which a dedicated jumphost is used and the standard ssh
proxying mechanisms are used to access the session via the jumphost;
- an approach in which a dedicated proxy is provided as part of the Renku
platform which provides ssh access without needing to use the ssh client
proxy capabilities.
Each of these approaches is discussed in more detail below.

Other solutions could be envisaged: it could be possible to provide a tunnel
over HTTP/S into the session and access it via ssh locally or perhaps some
wireguard solution might work but these approaches would probably encounter
issues with different OS versions, would not be easy to configure and
ultimately would likely have a detrimental impact on user experience and hence
are not considered further.

### Exposing SSH port from each session directly

In this approach, each user session exposes at least 2 ports: an HTTP port
for the Jupyter lab server and an SSH port for the SSH session. These ports
have corresponding services running inside the pod.

Services are then linked to these sessions with the ports on the services
mapping to the ports exposed by the pod.

In this solution, an ingress is mapped directly to the SSH services with a
dedicated ingress for each SSH session. Each ingress should have a unique name 
and also a unique IP address as the SSH protocol operates
on an IP address basis (i.e., it has no specific server identification such as
SNI which can enable a single server to handle requests for multiple endpoints). 
This solution can suffer from DNS propagation delays, depending on the DNS
provider and DNS configuration. Further, exposing ssh servers running inside
user sessions directly to the Internet brings some security risks and it would
need to be clear that users cannot easily change the ssh daemon configuration
within their sessions to make their session very exposed.

A couple of further points relating to this approach:
- ingresses on cloud providers can typically be expensive; having one per
  session is likely to incur significant cost;
- on the Switch deployments, the entire cluster is exposed via a single IP
  address; as such it is not straightforward to devise a solution in which
  different sessions would map to different IP addresses.

### Adding a proxy jumphost

Another solution is to have a jumphost and to use this to access the pods. In
this case, a jumphost must be installed inside the kubernetes cluster as it 
requires direct access to the ssh server running on the pods.

This requires modifications to the cluster ingress configuration to forward TCP 
traffic incident on a specific port to the jumphost pod on its SSH port; this
pod must have its sshd configured to support proxying. Also, as ssh proxying
approaches require authentication against both the proxy and the destination
server, an approach in which some combination of either no credentials and
a keypair for which the user possesses the private key must be performed.

The most natural solution in this case is to use the user's public key within
the session and have open access with no ability to obtain a shell within the
jumphost.

In this case, the command to ssh into a session requires specifying both the
jumphost and the destination; as such it is a more complex command and introduces
some user friction when accessing sessions. VSCode supports such proxying
scenarios without problem.

### Using a Man In the Middle Proxy solution

The third approach is the one followed by gitpod; in this approach a dedicated
proxy is provided which affords access to the sessions. As with the proxying
approach above, this must run on the kubernetes cluster.

Unlike the above approach, this requires writing and maintaining a software
component. The key difference is that this proxy does not use the proxying
capabilities provided by an SSH client in which one SSH session is embedded
inside another; effectively this approach concatenates two distinct SSH
sessions with two distinct authentication flows. The proxy terminates one
SSH session and forwards all traffic to another SSH session.

This solution has the following benefits:
- it is possible to log in via a simple ssh command which looks much cleaner/more
professional;
- there is a single entry point which reduces the attack surface;
- it is reasonably straightforward to add instrumentation such that SSH session
information data can be easily collected.

The primary downside is that it involves writing and maintaining another
component; however initial work by Tasko indicates that this can be done with
a modest amount of coding effort.

## Proposed Solution

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
