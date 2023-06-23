# Implementing a Spam Management Service for GitLab

- Start Date: 2023-05-19
- Status: Proposed

## Summary

This proposal aims to introduce a spam management service to mitigate spam on GitLab and improve user experience and security. This service will use machine learning models to detect and eliminate spam across the platform in real-time where possible, focusing on snippets, issues, notes, projects, and user profiles.

## Problem Statement

GitLab, a service on which RenkuLab relies, is not immune to the issue of spam. Spam can infiltrate various platform aspects, including snippets, issues, notes, projects, and user profiles. Spam disrupts the user experience, poses potential security risks, and damages the reputation and RenkuLab.io's SEO. GitLab lacks a robust, real-time solution to identify and eliminate spam across all attack vectors.

## Key Assumptions

- The proposed machine learning model will be built in consultation with the innovation team. The machine learning model's specifics are out of this RFC's scope.

## Possible Solutions

- A service that only checks for spam at regular intervals. This solution is less than ideal because of the delay between the spam being posted and the spam being detected and removed, during which spam could already be harmful. 

- Monitor GitLab for spam manually. This solution is unsuitable because it is time-consuming.

- GitLab has its [Spamcheck](https://docs.gitlab.com/ee/user/admin_area/reporting/spamcheck.html) service, which can be used to check for spam issues and snippets. This service does not check for spam in notes, projects, or user profiles. Currently, it is not possible to configure the service to delete spam automatically. Deploying GitLab's Spamcheck service also requires GitLab Enterprise Edition, which RenkuLab.io's GitLab is not currently running on.

- GitLab can make use of [Akismet](https://akismet.com/), a third-party-hosted AI spam filter. However, this only checks for spam on issues and notes, needs training, and would involve handing data over to a third party in the US.

## Proposed Solution

The proposed spam management service will integrate with GitLab via system hooks and regularly scheduled jobs. When notes, projects, or users are created or updated, GitLab will send system hooks to a spam management service. System hooks request examples can be found [here](https://docs.gitlab.com/ee/administration/system_hooks.html#hooks-request-example).

System hooks triggered on group, project, and user events can be triggered at the system level. Issues and comments at a project level would require a project-level system hook. This could be configured when a project is added to KG, or KG could forward project-level events to the spam management service's event service. Project-level webhook events examples can be found [here](https://docs.gitlab.com/ee/user/project/integrations/webhook_events.html).

Snippets do not have system hooks. Therefore a job triggered at regular intervals will be required to check for new snippets. This can be done by checking the snippet API for new snippets. Documentation on the snippet API can be found [here](https://docs.gitlab.com/ee/api/snippets.html).

Ideally, the solution will also be able to check abuse reports manually submitted by users in GitLab and check the reported URL in the abuse report for spam content.

A feedback mechanism would also be implemented, allowing the machine learning model to learn from mistakes and improve over time. This could be done by storing content in an S3 bucket and regularly re-training the model on a dataset including new data. The system should also keep a log of decisions made and the confidence level of the decision, as well as export Prometheus metrics which can be used to compile regular reports.

Similarly to GitLab's Spamcheck service, storing the model in an S3 bucket and pulling it to the spam management service on start-up would allow the model to be updated without having to redeploy the spam management service.

An allowlist of trusted email domains could be implemented to prevent false positives and ensure users with trusted email domains can post content without being checked for spam. YAT already possesses a list of trusted email domains. Users without trusted email domains could also be verified with a 'verified' string in the GitLab user's admin note. A list of verified users would be stored in Redis and updated periodically. This would allow users to be verified without having to redeploy the spam management service.

A queue management system such as [Redis lists](https://redis.io/docs/data-types/lists/) would be used to track items that need to be classified and spam items that need to be deleted from GitLab. This helps to ensure the system can handle many system hooks from GitLab without significant latency and without overloading the GitLab API. Redis was chosen because of GitLab's existing reliance on Redis; and not wanting to add another dependency to the system, which should be able to work independently of RenkuLab. Depending on the amount of data sent to the service and its efficiency, it may be necessary to implement Redis persistence in the case of a crash.

This mermaid diagram provides an overview of how this system would work:

```mermaid
  flowchart TB
    subgraph "GitLab Components"
      GH[GitLab System Hook]
      API[GitLab API]
    end

    subgraph "Redis Components"
      Q1[GitLab Item Fetch Queue]
      Q2[User Trust and Verification Queue]
      CQ[Classification Queue]
      DQ[Deletion Queue]
      VRU[Verified Renku Users List]
    end

    subgraph "Spam Management System"
      subgraph "Jobs"
        SC[Snippet Check Job]
        VU[Verified Users Retriever Job]
      end
      subgraph "Services"
        ES[Event Service]
        FS[GitLab Item Fetch Service]
        TV[User Trust and Verification Service]
        CS[Classification Service]
        DS[Deletion Service]
        subgraph "Other Components"
          S3[S3 Bucket]
          L[Logs]
          TED[Trusted Email Domains List]
          ML[Classifier Model]
        end
      end
    end

    GH -->|Triggered on item create/update| ES
    SC -->|Runs Periodically| ES
    VU -->|Updated Periodically| VRU
    ES --> Q1
    Q1 --> FS
    FS -->|Get Item| API
    FS -->|Add Item To Queue| Q2
    Q2 --> TV
    TED --> TV 
    VRU --> TV 
    VU -->|Get Verified Users| API
    TV -->|If Not Trusted Or Verified| CQ
    CQ --> CS
    CS -->|If Item Is Spam| DQ
    CS -->|Classification Result| L
    DQ --> DS
    DS -->|Delete Item from GitLab| API
    DS -->|Set Item to Private in GitLab| API
    CS -->|Store Evaluated Content| S3
    S3 -->|Download Model On Startup| ML
    CS <-->|Classification| ML
    
    style GH fill:#E24329,stroke:#333,stroke-width:4px
    style API fill:#E24329,stroke:#333,stroke-width:4px
    style ES fill:#645EB6,stroke:#333,stroke-width:4px
    style SC fill:#E46991,stroke:#333,stroke-width:4px
    style FS fill:#645EB6,stroke:#333,stroke-width:4px
    style TV fill:#645EB6,stroke:#333,stroke-width:4px
    style CS fill:#645EB6,stroke:#333,stroke-width:4px
    style DS fill:#645EB6,stroke:#333,stroke-width:4px
    style VU fill:#E46991,stroke:#333,stroke-width:4px
    style Q1 fill:#009A6A,stroke:#333,stroke-width:4px
    style Q2 fill:#009A6A,stroke:#333,stroke-width:4px
    style CQ fill:#009A6A,stroke:#333,stroke-width:4px
    style DQ fill:#009A6A,stroke:#333,stroke-width:4px
    style VRU fill:#009A6A,stroke:#333,stroke-width:4px
```

Decision tree for when an issue is created on GitLab:

```mermaid
  graph TD
    IC[Issue Created] --> GH[GitLab System Hook]
    GH --> ES[Event Service]
    ES --> FS[GitLab Item Fetch Service]
    FS --> TV[User Trust and Verification Service]
    TV --> E{User has Trusted Email Domain?}
    TED[Trusted Email Domains List] --> E
    E --> |Yes| X[End: Issue Allowed]
    E --> |No| F{User is Verified?}
    VRU[Verified Renku Users List] --> F
    F --> |Yes| X
    F --> |No| CS[Classification Service]
    CS --> SC{Classified as Spam?}
    SC --> |No| X
    SC --> |Yes| DS[Deletion Service]
```

Decision tree for periodic snippet checking:

```mermaid
  graph TD
    A[Periodic Snippet Check] --> ES[Event Service]
    ES --> FS[GitLab Item Fetch Service]
    FS --> D[For Each Snippet]
    D --> TV[User Trust and Verification Service]
    TV --> E{User has Trusted Email Domain?}
    TED[Trusted Email Domains List] --> E{User has Trusted Email Domain?}
    E --> |Yes| X[End: Snippet Allowed]
    E --> |No| F{User is Verified?}
    VRU[Verified Renku Users List] --> F
    F --> |Yes| X
    F --> |No| CS[Classification Service]
    CS --> SC{Classified as Spam?}
    SC --> |No| X
    SC --> |Yes| DS[Deletion Service]
```

## Drawbacks

1. Ensuring the system can handle many system hooks from GitLab without significant latency is critical. It needs to be scalable and able to handle peak loads.
2. The spam management service will require a GitLab personal access token with administrative rights to delete spam.
3. We must ensure we can be informed of and restore incorrectly classified content.

## Rationale and Alternatives

This solution is deemed the best as it provides real-time spam detection without intruding on the user experience. Alternatives such as manual monitoring are too time-consuming.

Not implementing this solution would mean continuing to expose GitLab users to potential spam, which could disrupt their experience and pose security risks, and open RenkuLab.io up to promoting unethical content.

## Unresolved Questions

- How will user privacy be protected? Storing datasets with legitimate content in an S3 bucket might be unsafe.
- How and when will users be notified if their content is identified as spam and removed?
