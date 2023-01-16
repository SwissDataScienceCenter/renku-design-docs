- Start Date: 2023-01-13
- Status: Accepted*

(This is marked as Accepted as it has been agreed in different contexts at
different times; it did not follow the standard RFC process per se - rather
much of the content is taken from a lengthy PR discussion and stored here for
reference.)

# Use of RabbitMQ within Renku

## Summary

> One paragraph explanation of the change.

As of this writing, interaction between Renku services is done via HTTP
requests; this means that all interactions are essentially synchronous -
further, a synchronous modus is used for some communications which are
fundamentally asynchronous, eg polling mechanisms are used to obtain state
updates. Adding messaging functionality has been on the wish list for some
time. Here, we describe how RabbitMQ will be added to Renku, what services will
use it and provide some information on their requirements.

This document essentially captures a discussion which took place on (this issue)[https://github.com/SwissDataScienceCenter/renku/pull/2872].

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

Support for messaging capabilities is a natural requirement in a microservices
context, especially when there are some operations which may take an unknown
amount of time. Ralf has performed a survey of different messaging solutions
and concluded that RabbitMQ is the best fit with the needs of Renku.

As well as this general driver, there are a number of specific use cases which
can benefit from the availability of a messaging solution - these are:

- the ui-server should be notified of session state changes by renku-notebooks;
  this is currently done via a polling mechanism but a messaging solution would
  be neater
- the core-service currently runs long-running jobs such as project migration
  - migrating from one Renku project version to another - and dataset import using
  python-rq. It would be desirable to realize both of these issues in a more
  async manner such that a notification is triggered when the job is complete.
- the Knowledge Graph currently uses the Renku CLI to generate triples; this is an
  undesirable legacy dependency. A better way for this to work is for the
  core-service to post the job to the message bus, it gets picked up by the
  triples generator which posts a message to the message bus when finished.
- In a similar vein, using an async approach, it could be possible to change
  the order of triples processing such that a user who is using the web UI could
  cause relevant triples to be generated immediately rather than waiting for them
  to be processed in order, thus improving their user experience.
- KG can notify UI when triples are processed/a project is picked up by the KG,
  instead of the UI polling
- In future, it will be desirable to support user workflows run in a Renku context -
  in this case, having some a messaging oriented solution for triggering jobs, 
  monitoring  progress and knowing when they have completed will be required.

## Design Detail

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the infrastructure to understand. This should get into specifics and corner-cases,
and include examples of how the service is used. Any new terminology should be
defined here.

The requirements are still imprecise; as such there is no specific design
for a messaging solution, as yet. In the issue noted above, the following points
relating to requirements were flagged:

- it is anticipated that a message throughput of max some 10's of messages/sec
  is required
- persistence requirements at least initially will be minimal
  - Sean suggested introducing some persistence now with the idea that it's easier
    increase it rather than add it from scratch
- there should be minimal sensitive content on the message bus
- it may be necessary to expose the message bus to the `renku` cli; in this
  case an OAuth2 mechanism provided by Keycloak could be appropriate

## Drawbacks

> Why should we *not* do this? Please consider the impact on users,
on the integration of this change with other existing and planned features etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Rationale and Alternatives

> Why is this design the best in the space of possible designs?

> What other designs have been considered and what is the rationale for not choosing them?

> What is the impact of not doing this?

## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

> What parts of the design do you expect to resolve through the implementation of this feature before stabilisation?

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
