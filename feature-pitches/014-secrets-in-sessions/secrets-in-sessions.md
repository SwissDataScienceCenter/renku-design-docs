# Bringing secrets to sessions

Authors: Rok Roškar, Laura Kinkead

> [!IMPORTANT]
> We are building the next version of Renku - [Renku 2.0](https://blog.renkulab.io/renku-2)! Would
> you like to get involved in shaping the future of Renku? Interested to participate in our user
> research? Get in touch! hello@renku.io

## 🤔 Problem

Users would like to configure secrets to be injected into their session environments. Currently we
have a service for storing secrets but they are not yet used anywhere.

Users want to use secrets in sessions to be able to:

- connect to a database
- reach further services safely (storage, HPC, etc).
- Below are from [PR: Facilitated storage compute
  access](https://github.com/SwissDataScienceCenter/renku-design-docs/pull/29]:
  - query a large private data archive to discover and select datasets
  - fetch small datasets from the private data archive and explore them interactively
  - submit processing of the datasets to private computing cluster storing the results in private
    storage
  - fetch some results to interactive session, explore and visualize them

## 🍴 Appetite

3 weeks

## 🎯 Solution

> [!NOTE]
> 🪓 This pitch is scoped for only current Renku sessions, not Renku 2.0 sessions. We do
> want this functionality in Renku 2.0, but it’s not in scope for this pitch.

We can keep this simple - we require the creation of a new *beta* secrets settings page and a new
section in the “launch with options” that gives the user the option to choose secrets to use in the
session.

### 🚞 User stories / journeys

#### Defining a secret

1. a user navigates to a new settings page for secrets
2. they can see, but not edit, existing secrets on this page
3. they are able to add a new secret key/value pair
4. they can also delete existing secrets

#### Using a secret

1. No secrets are included by default in sessions —> this is a beta feature
2. They can be added on the “Launch with options” page, just like e.g. storage. The user can choose
   whether to use the secret as an environment variable or have it mounted on the filesystem at e.g.
   `/secrets`
3. Secrets are entirely user-scoped, i.e. each user needs to manage their own and there is no such
   thing as a project secret
4. The secret gets injected into the session - either as an environment variable or mounted as a k8s
   secret and accessible on the filesystem

## 🐰 Rabbit Holes

Care should be taken to make sure that this implementation can be extended to use secrets in the
storage service in the future.

## 🙅‍♀️ No-gos

Using Secrets in the storage service, though this is something we definitely want eventually!
