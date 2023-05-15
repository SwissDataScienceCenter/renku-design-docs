- Start Date: 2023-05-08
- Status: Proposed

# Facilitated Access to Project Storage and Compute Resources

## Summary

> "One paragraph" explanation of the change.

In modern astrophysical projects it is often necessary to access astronomical data archives (S3, various block storage, WebDAV, etc) and computing (e.g. dask, ARC, slurm, reana, etc) services. Most of the time, the access is through an interface built upon HTTP and relying on OpenID Connect for access control.
One of the recongised benefits of interactive data analysis and exporation platforms like jupyterhub (and, hopefully, renkulab) is that it is possible to facilitate access to these resources in the following ways:
* Interactive platform may allow users to discover these services, with various complementary UI features. 
* Since access to these resources is typically restricted, it is possible to rely the identity of the users logged in the interactive session to enable access to the resources without any additional authentication.
* Often these resources can be located close to the platform, saving on data transfer costs, enabling the "bright code to the data" approach.

## Problem Statement

> Why are we doing this? What use cases does it support? What is the expected outcome?

This will enable users of astronomical infrastructures to easily access their data archives and computing clusters from renkulab, making renkulab much more attractive for them. 

A typical data-intensive astronomer life cycle can be outline like so:

* query a large private data archive to discover and select datasets
* fetch small datasets from the private data archive and explore them interactively
* submit processing of the datasets to private computing cluster storing the results in private storage
* fetch some results to interactive session, explore and visualize them 

In this process, renkulab can act as an interactive platform. The resources can be accessed through an API. Forumulation of the request and substitution of authorization can be facilitated by renkulab.

Some example services include:

* storage:
    * WebDAV storage with OIDC or OIDC-to-certificate mapping
    * [rucio](https://github.com/rucio/rucio) bulk archive
    * CSCS [object storage](https://user.cscs.ch/storage/object_storage/) 
* compute
    * EGI/[ARC](http://www.nordugrid.org/arc/) cluster
    * [DIRAC](https://github.com/DIRACGrid/DIRAC) cluster
    * custom compute MMODA API https://www.astro.unige.ch/mmoda/


> Can we distinguish between essential requirements and 'nice to have' requirements

Nice to have feature would be to add features in the session UI to make it easier to start connection to the storage and compute services. For example, a button "fetch data from ESAC archive" or "run in Pulsar Network". Something similar is done in dask jupyterlab plugin.

It could be also interesting to present the archives as mounted filesystems (it is done like that in ESA DataLabs and also ESCAPE/ESAP).

## Key Assumptions

> Are there any key assumptions underpinning this work which should be highlighted?
These could be assumptions on user behaviour, assumptions on the deployment
context, assumptions on external services etc

This requires depenency on external compute and storage services. There might be testing to make sure these services remain functional.

## Possible Solutions

> In most cases in which an RFC is required, there is not a single obvious solution
and it makes sense to itemize a few different alternatives/approaches. If that is
the case, these should be included here.

***It can make sense to organize a meeting at which the Problem Statement, Key
Assumptions and Possible Solutions are discussed to explore the problem space
as a lead in to defining the Proposed Solution.***

### A

A possible soluton might be similar to "jupyterhub services" - specialized services deployed along with a jupyterhub instance. Jupyterhub provides a framework for sharing user account with these services.
If similar solution is available in renkulab, it would be possible outsource development of specialized domain-specific compute and storage interfaces to domain-specific teams.

Renkulab would need to set some environment variables before the session start, specifying the service endpoints and possibly credentials.

### B

 Another possible solution could be to allow users to access external compute and storage services directly from renku session without any additional domain-specific services near renku, using renku to store authorization for accessing external services associating it with user account. Renkulab would then set some environment variables for the session, and they will be used by the user code.

## Proposed solution

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody familiar with the 
infrastructure to understand. This should get into specifics and corner-cases, 
and include examples of how the service is used. Any new terminology should be 
defined here.

***It can make sense to organize a meeting to discuss the Proposed Solution such 
that the proposal is clear, the solution is convincing and the team believes it
can be implemented in a reasonable amount of time.***

## Drawbacks

> Why should we *not* do this? Please consider the impact on users,
on the integration of this change with other existing and planned features etc.

Interfaces to storage and compute backends proposed in this RFC would be specialized, useful to different individual projects, communities. In principle this may lead to numerous project-specific and domain-specific interfaces which will be difficult to support on single renkulab instance. 

Second possible solution, just storing the credentials, is more favorable in this respect. It still implies, however, that renkulab is keeping track of multiple domain-specific external services and of the way the can be made available to the user session. So a lightweight "renkulab services" contributed by teams may still be benefitial to separate responsibility for these domain interfaces.

> There are tradeoffs to choosing any path, please attempt to identify them here.



## Rationale and Alternatives

> Why is this design the best in the space of possible designs?

> What other designs have been considered and what is the rationale for not choosing them?

> What is the impact of not doing this?

Right now, when we need to access secured external resoure from renkulab, we need to manually copy the credentials into the session, and make sure they are not stored in the repository which would compromise them. This is cumbersome and potentially unsafe.

When transferring large datasets to renkulab, we need to take care of transfer costs ourselves. It means that even moderately large data is usually not transferred to renkulab.

## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process 
before this gets merged?

> What parts of the design do you expect to resolve through the implementation 
of this feature before stabilisation?

> What related issues do you consider out of scope for this RFC that could be 
addressed in the future independently of the solution that comes out of this RFC?
