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

This document essentially captures a discussion which took place on [this issue](https://github.com/SwissDataScienceCenter/renku/pull/2872).

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

Support for messaging capabilities is a natural requirement in a microservices
context, especially when there are some operations which may take an unknown
amount of time. Ralf has performed a
[survey](https://docs.google.com/spreadsheets/d/138apZ6QOLIkUSdiyViGVYbCMIb9-1RX20b0wJXJpiSc/edit?usp=sharing)
of different messaging solutions and concluded that RabbitMQ is the best fit
with the needs of Renku.

As well as this general driver, there are a number of specific use cases which
can benefit from the availability of a messaging solution - these are:

- the `ui-server` should be notified of session state changes by `renku-notebooks`;
  this is currently done via a polling mechanism but a messaging solution would
  be neater;
- the core-service currently runs long-running jobs such as project migration - 
  migrating from one Renku project version to another - and dataset import using
  `python-rq`. It would be desirable to realize both of these issues in a more
  async manner such that a notification is triggered when the job is complete;
- the Knowledge Graph (KG) currently uses the Renku CLI to generate triples;
  this is an undesirable legacy dependency. A better way for this to work is
  for the core-service to notify KG probably via a synchronous call, KG then
  posts the job to the message bus, it gets picked up by a triples generation
  process which does its work and posts the resulting triples to the message
  bus when finished. Specifics around how this can work have not been agreed;
  one idea would be to use a workflow service such as `argo`, another approach
  could be to simply have workers which scale with load;
- currently, the UI polls the Knowledge Graph for status of processing of
  triples; an async, messaging based  solution in which the UI was made aware
  of such processing would be more desirable than a polling based approach;
- in future, it will be desirable to support user workflows run in a Renku context -
  in this case, having some a messaging oriented solution for triggering jobs, 
  monitoring progress and knowing when they have completed will be required;
- a use-case in which a queue controls write access to jena could be considered;
  as jena is not transaction oriented, it would be nice to have a single writer
  which would take work from a queue and write to jena. As such, the triples
  generators would make their output available, post to the queue and the writer
  would take this info from the queue and write in an ordered fashion. This
  could simplify the operation of the KG.

## Design Detail

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the infrastructure to understand. This should get into specifics and corner-cases,
and include examples of how the service is used. Any new terminology should be
defined here.

The requirements are still imprecise; as such there is no specific design for a
messaging solution as yet. In the github issue noted above, the following
points relating to requirements were flagged:

- it is anticipated that a message throughput of max some 10's of messages/sec
  is required with something of the order of 200-500 concurrent users;
- persistence requirements at least initially will be minimal
  - Sean suggested introducing some persistence early with the idea that it's easier
    to increase it rather than add it from scratch when we are in an operational context;
- there should be minimal sensitive content on the message bus;
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


Ralf considered different messaging solutions
[here](https://docs.google.com/spreadsheets/d/138apZ6QOLIkUSdiyViGVYbCMIb9-1RX20b0wJXJpiSc/edit?usp=sharing) - the contents are included here for completeness.

|                                         | RabbitMQ                                           | Pulsar                                          | NATS                                    | Kafka |
|-----------------------------------------|----------------------------------------------------|-------------------------------------------------|-----------------------------------------|-------|
| Throughput (m/s)                        |                                         35k (12k*) |                                            100k |                            200k+(100k*) | 200k+ |
| Throughput (mb/s)                       |                                             50-100 |                                             300 |                                500(50*) |   600 |
| Latency (ms) @ 10k/s                    |                                            17(27*) |                                              14 |                                      20 |  4-10 |
| Latency (ms) best                       |                                                  3 |                                               7 |                                       2 |     3 |
|                                         |                                                    |                                                 |                                         |       |
| Features:                               |                                                    |                                                 |                                         |       |
| Persistence(crashes)                    |                                                  ✓ |                                               ✓ |                                       ✓ |       |
| Persistence(Long Term /for replay)      |                                                ✓** |                                               ✓ |                                       ✓ |       |
| AuthN                                   |                Passwd  X.509 OAuth2(Keycloak) LDAP |        HTTPBasic Oauth2/JWT TLS Kerberos Athenz |       Passwd Token TLS NKEY JWT(custom) |       |
| AuthZ                                   |                        vhost:Exch\|Queue:Topic JWT |                             namespace/topics(?) |             subjects (replies) Accounts |       |
| Reliability                             |          at least once at most once (exactly-once) |                      at least once exactly once | at least once at most once Exactly-once |       |
| Congestion/Buffers                      |                         prefetch(QoS) flow control |                                        prefetch |                                prefetch |       |
| Protocols                               | AMQP STOMP MQTT RMQ Streams Websockets(STOMP,MQTT) |                                          custom |      custom MQTT Websocket (Connectors) |       |
| Deadletter                              |                                                  ✓ |                                               ✓ |                                       x |       |
| Transactions                            |                             ✓ Single Queue/Message |                                               ✓ |                                       x |       |
| Prometheus                              |                                                  ✓ |                                               ✓ |                                       ✓ |       |
| Clients                                 |                  Python Java Javascript/Node Scala |                             Python Java Node.js |                     Python Java Node.js |       |
| Streams                                 |                                                  ✓ |                                               ✓ |                                       ✓ |       |
| Helm Chart with Clustering  And Scaling |                                                ✓ ✓ |                                             ✓ - |                                    ✓ ✓* |       |
| Multi-Tenancy                           |                                                  ✓ |                                               ✓ |                                       ✓ |       |
| Routing                                 |                        direct fanout headers Topic |                                           topic |                                   topic |       |
| Resources                               |              256mb Mem min (5node, 16 vCPUs, 32GB) |                     6 Linux machines 8 CPU, 8GB |                   1 CPU, 64MB (3 Nodes) |       |
| Argo                                    |                                                  ✓ |                                               ✓ |                                       ✓ |       |
| Docs                                    |                                                5/5 |                                             3/5 |                                     4/5 |       |
| Extras                                  |                                                    | Functions IO(Solr) SQL Encryption Deduplication |                   KV-Store Object Store |       |
| Components                              |                                             1 (3+) |                                            5-8+ |                                 1 (3/5) |       |
|                                         |                                                    |                                                 |                                         |       |
|                                         | *classic queues, (otherwise quorum queues)         |                                                 | *jetstream                              |       |
|                                         | **With “streams” feature                           |                                                 |                                         |       |


## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

> What parts of the design do you expect to resolve through the implementation of this feature before stabilisation?

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
