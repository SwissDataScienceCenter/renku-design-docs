# Project workflows UI view

Created: July 6, 2023 3:18 PM
Tags: CLU, UI, storage

---

## 🤔 Context and Problem

We want Renku to easily provide access to compute and data. However, data access
in Renku has often been somewhat complicated, and we would like to alleviate
this problem.

### LFS as a default

Users struggle with data in Renku. The default way of handling data is through
git-LFS, which has many nice properties like automatically integrating with the
git workflow, versioning of data etc. It also has many drawbacks like requiring
double space for data (once in the cache, once in the tree), “locking” data into
a repo (can’t access it any other way), etc. Using git-LFS for small-ish data
and for e.g. results makes sense - using it for large data sets that rarely
change and don’t need to be versioned, does not. Using data through git-LFS also
requires it to be downloaded every time a user starts a session, which leads to
huge overheads for projects with lots of data.

### External storage in sessions

In addition to the above “external” data sources for Renku Datasets, it is
possible to add an S3 bucket or Azure Blob Storage mount to a user session. The
UX around this is currently pretty poor, as the user needs to enter bucket
information every time they launch a session. Furthermore, there is no
connection between these mounts and potential usage of this data in a Renku
project.

### User stories

1. A small team of data scientists is collaborating with domain scientists who
   provide the raw data, which is in the TB size range. The data is not
   changing, but more data is added periodically. The data scientists manipulate
   the data into a format appropriate for training ML models and want to share
   the results easily with their domain scientist counterparts. They use a mix
   of remote (renku) and local (laptop or workstation) resources to work on the
   data and collaborate. (concrete use-case from the academic team)
2. Data science team has data in S3 → they don’t want versioning, want access to
   the same data in the same project and have it mounted automatically (see [FSO
   Dashboards User
   Research](https://www.notion.so/FSO-Dashboards-User-Research-fe67bc232647489690954f24b134811a?pvs=21)).
   They don’t care about publishing, persistence, reproducibility, just want
   access to data and they want to be able to write back to the bucket.

### Summary

There are several issues that need to be resolved:

- The disconnect between “storage” in projects, sessions and datasets
- Lack of information in the UI that a dataset is backed by cloud storage
- Friction in making data from cloud storage available in interactive sessions

## 🍴 Appetite

6 weeks. This is essential and needs a solid implementation. Keep in mind that
this implies 5+1 weeks, as a week will certainly be needed to finalize and
polish deployment / presentation / QA.

## 🎯 Solution

The solution presented below has many parts. The progression in which the
various parts should be considered is as follows:

1. Project-level storage with automatic mounting in sessions
4. Credentials storage for seamless mounting in interactive sessions and from
   the CLI

We should consider project-level storage to be *required* and the credentials
storage potentially a part of a second effort. Datasets will not be considered
in this build but will be added after we gain some experience with the initial
implementation.

### Defining “storage” for a Project

Storage sources should be a part of the high-level Project configuration. For
example, we could imagine commands like

```bash
$ renku storage add
$ renku storage ls
$ renku storage mount
$ renku storage umount
```

In the Renku web UI, project cloud storage should be configurable from the
project settings page, using an endpoint on the core service.

`add` is used to define a new storage at the project level and in its simplest
form, takes the URL of the remote storage as argument plus the target folder
where it should be mounted to. Providers can add additional options to manually
specify fields required for the storage.

`ls` lists all storage that is configured for the current project.

`mount`/`unmount` mount resp. unmount either all or the specified storage in the
project.

### Session launch

If *any* external/cloud storage is configured for the project this data should
*by default* be automatically mounted when the session is launched.

****UI****

We may want to offer the option to *not* automatically mount storage (the user
might prefer to copy the data). We should assume that credential storage *will
exist* for the purpose of this feature, but we should consider cases where the
credentials are missing or invalid. In such cases, we need to be able to prompt
the user for new credentials or offer to launch a session without the storage
attached. For this, we may need to interrupt the session launch (i.e. the case
where someone clicks the “play” button) to ask for credentials — ideally we
wouldn’t fully stop the launch but just ask for credentials during the
pre-flight check.

******CLI******

In the CLI we can easily persist the credentials safely and send them along with
the session launch request. Just as in the UI case, we should prompt the user
whether they want to mount the storage (and offer a flag to circumvent the
prompt). Credentials could be obtained from the credential store or from a local
config; just like in the UI case above, we should verify that they actually work
during the pre-flight check. If they don’t work, prompt the user to enter them
or offer the option of *not* mounting the data.

### In the metadata

The storage backend information has to be added to the Project metadata and the
KG needs to return this information so that clients can access it.

The metadata should be stored as nested objects in the `Project` entity inside
the renku-python metadata.

The [Renku Ontology](https://swissdatasciencecenter.github.io/renku-ontology/)
gets extended as follows:

`schema:Project` gets extended with a property `renku:hasStorage` that points to
one or more `renku:RemoteStorage` entity. Each `renku:RemoteStorage` has an
`id`, `provider` and `url` field. Further fields may be added in the future,
especially storage provider specific ones, but for now the URL should be enough
to define a storage (see below). The id follows the form
`https://<renku-instance-host>/storage/<provider>/<host>/<bucket|container>` .
E.g. `https://renkulab.io/storage/s3/amazonaws.com/giab`.

`schema:Project` gets extended with an additional property `renku:mountpoints`
that points to one or more `renku:StorageMount` which consist of an `id`,
`renku:storage` pointing to the `id` of a `renku:RemoteStorage` and
`renku:mountpath` containing a string of where in the project the storage should
be mounted/copied to.

Note that mount paths and storage are defined separately on the project, since
different folders from the same storage might be mounted to different folders
and to allow potential reuse of storage between projects with different mount
paths.


### Valid storage URIs

#### S3

- s3://\<bucket\>/\<path\>  (uses default region on AWS)
- (s3|https)://s3.\<region\>.amazonaws.com/\<bucket\>/\<path\>
- (s3|https)://\<bucket\>.s3.\<region\>.amazonaws.com/\<path\>
- (s3|https)://\<host\>/bucket/\<path\> (for third party providers)

#### Azure Blob Storage

- (az|azure)://\<container\>/\<path\>
- (az|azure)://\<account\>.dfs.core.windows.net/\<path\>
- (az|azure)://\<account\>.blob.core.windows.net/\<path\>

The following are not supported for now:

- adl://\<container\>/\<path\> (according to fsspec)
- abfs://\<container\>/\<path\> (according to fsspec)
- abfs://\<file_system\>@\<account_name\>.dfs.core.windows.net/\<path\>


#### Google Cloud Storage

Support for this is optional. The only format supported is:
gs://\<bucket\>/\<path\>


### Credentials

We should aim to have a service in place to store and serve user credentials for
these features. However, this might not be feasible in the amount of time we
have for the pitch. In that case, the acceptable compromise is to a) ask for
credentials in the UI on session launch (or dataset manipulation) and b) do the
automatic credential forwarding from the CLI only.

## 🐰 Rabbit holes

- Performance: we know that mounting data sources directly might not be the most
  performant option; we should focus on usability over performance for the time
  being and think about optimization later (we could imagine creating cached
  copies of data on high-performance storage mounts, for example)re

## 🏅 Nice to haves

- Secret/credential storage - we should consider how difficult it would be to
  deploy a service responsible for handling user secrets. One may already be
  available off-the-shelf

## 🙅 Out of scope

- be able to mount buckets in an active session, not only at launch
  - requires sidecar (Tasko), security concerns
- User-level defaults that would apply / be available in different contexts,
  e.g. to define some preferred storage locations as a user and offer to mount
  them in any arbitrary session. We should focus for now on Projects.
