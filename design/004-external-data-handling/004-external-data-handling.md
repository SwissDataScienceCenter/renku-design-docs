- Start date: 30.03.2022
- Status: Draft

# External Data Handling RFC

## Summary

A framework for handling external data (S3, NFS, etc.) is described.

## Motivation

The default mode of data management in Renku is via git and git-lfs. This is
fine for uncomplicated projects with modest data needs, but often it is neither
the most appropriate nor the most efficient solution. Primarily, many users want
to use data from external sources without needing to transfer it to Renku
servers for storage. Renku should not get in the way of interacting with
external data sources, and should facilitate access and abstract away
boilerplate as much as possible.

This RFC describes the general framework for how Renku should support data from
external sources. The primary concern to address is how to do justice to
external data with respect to provenance tracking; Renku should preserve the
information about the original source of the data and provide mechanisms to
retrieve this data when necessary.

### User stories

#### 1. Sharing data for ad-hoc analysis

This example uses S3 but it could be nearly identical with e.g. an institutional
NFS server or something like openBIS.

Bob is a scientist working in a large international collaboration that relies on
sharing raw data that is very valuable, large (TB-range), and immutable. Bob
starts looking at the data on his laptop. The data is in an S3 bucket that his
lab uses. He uses s3fuse to mount the S3 bucket with the data to his laptop.

He then decides that he needs to give others access to his work-in-progress so
they can decide on future direction for the analysis. He wants people to have
access to the same compute runtime so there are no conflicts with libraries that
are used. He creates a renku project on his laptop, updates the list of
dependencies and creates a dataset. He adds the data from S3 to his Renku
dataset; Renku queries him for credentials to be able to access the protected S3
bucket and they are stored in his local user configuration so he doesn't have to
enter them again. Bob uses the Renku CLI to mount the data to a directory in
his project so it can be used seamlessly like any other data or file that he
might have. He uses the data in his project to prepare some simple sanity checks
to share with his collaborators. He annotates the dataset with metadata so they can
more easily find it later if needed for another project.

Bob pushes the project to renkulab.io and shares it with his peers. The images are
built automatically and include the software packages needed for the analysis that
Bob created. He and his colleagues can launch interactive sessions for the project
and Renku automatically prompts them to mount the data from S3 that is used
in the dataset. Once the session starts, the S3 bucket is mounted automatically and
it is available in the same location in the project as it was before on Bob's
personal laptop.


#### 2. Tracking the use of external data

Sally is Bob's collaborator who continues to build on Bob's initial work by
building a more robust workflow. She uses standard Renku commands to record the
workflows. Renku uses the same mechanisms to attach the data as described above
when workflows are being re-executed or managed. She can also request to check
whether the results are still in sync with the data using `renku status`. In
this case, Renku checks the object hashes that were recorded previously against
the data currently found in the remote location (this might take a while over
the network - some data sources might provide pre-computed hashes). If the
hashes differ, Renku offers to update the results.

#### 3. Data storage for research project

Willow is interested in a large, protected dataset that is offered for free to researchers by an external organization. She has to accept a data access agreement to access the data, at which points she receives credentials to the bucket. To explore the dataset, she creates a Renku project and mounts the S3 bucket using the credentials.

After some exploration of the dataset, Willow decides to start a research project with the dataset. She makes a copy of the dataset in her own private S3 bucket for two reasons: (1) to make sure she maintains access to the data independent of the external organization, and (2) because she prefers to keep the project’s raw, intermediate, and final data products in the same place. She creates a Renku project for her work and mounts the bucket with her credentials. Throughout the research project, she reads the raw data from the mounted directory and saves intermediate data files and figures in separate folders in the same S3 bucket.

### Current status

#### Git-LFS

The Renku CLI uses git-LFS to store data alongside code in a repository. This is
pretty convenient because it allows us to use usual git semantics with data,
which means it can automatically be versioned and treated like other files in
the repository. The data normally gets pushed to a separate server, in this case
an object store configured alongside the GitLab server. However, it could be any
server supporting the git-LFS protocol. The Renku CLI includes git hooks that
prevent users a) from pushing data as normal git objects (which typically breaks
repositories), as well as b) from pushing normal code files (e.g. notebooks) into
git-LFS when not needed. This works fairly well but it is brittle - it's very
easy for users to misconfigure their repository and break repositories, even
when well-trained and well-intentioned.

An additional drawback of this approach is that data is currently always fetched
from the git-LFS object store whenever a repository is cloned. For some projects
this results in very long launch times for interactive sessions on renkulab.

#### "External" dataset data

The Renku CLI dataset functionality supports adding data from the filesystem
that is outside of the repository. For example

```shell
renku dataset add --external ../../path/to/my/data
```

The path to the data is stored in the dataset's metadata; the pointer can also
be updated in case that the data changes and the hash needs to be recomputed,
for the purposes of reproducibility. Of course, the obvious drawback here is
that the data location is arbitrary - if the project is brought to another
environment/machine/cluster, it's fairly likely that the data path will be
incorrect and therefore all recorded workflows will fail.

#### S3

Using S3 in Renku is currently limited to interactive sessions. When launching a
session, users can specify any S3-compatible URL together with credentials, if
necessary. The S3 bucket is then mounted in their interactive session.
Currently, no metadata about these mounts is recorded. The data could, in
principle, be added to a renku dataset with the "external" mechanism described
above, but no metadata about its actual origins (i.e. an S3 bucket) would be
recorded.

#### Upcoming

We have plans to include other external data providers, such as NFS shares,
openBIS etc. From the user's perspective, they all will generally work in the
same way as S3; data will be mounted to a local directory outside the repository
and made accessible as network-mounted storage.

## Design Detail

Generically, we should strive to remove the separation between "hosted" Renku
(i.e. what I can do if I run an interactive session on renkulab.io) and
"offline" Renku (what I can do with the CLI anywhere).

Ideally we could once again follow the plug-in paradigm in the CLI, like we
do for other functionality.

### Provider interface

If we adopt the view that data access should be done on the filesystem level,
then one approach would be to rely on FUSE as the interface. This means using
existing FUSE implementations where possible (e.g. S3) or implementing new ones
using e.g. [`python-fuse` library](https://github.com/libfuse/python-fuse).


### Local vs. hosted interactive sessions

Relying on FUSE for providing filesystem access to external resources means that
locally we do not require special privileges and can manage the data access for
the user. In the docker-based remote sessions, the situation is more complicated
because containers require elevated privileges to access FUSE on the host.
Therefore we will need to expose a microservice that will provide management of
external data sources for the user. The renku CLI would be aware that it is
running in a hosted session and instead of executing a local command for
mounting the data source, it would send off an API request to the service. The
service itself could run in a sidecar or be integrated with datashim (?), which
is currently used for providing S3 buckets in interactive sessions. If done via
a sidecar, we could likely achive dynamic mounting of data sources, which would
be fantastic for data exploration.

### Commands

#### Adding external data:

The examples below are based on the existing `dataset` commands, but we might
need to envision something new like `renku data source` which the `dataset`
commands could reference.


```
renku dataset add s3://server/path/to/bucket
Access Key: 1234
Secret Key: abcd
Store credentials? (y/N): y
Credentials stored in ~/.renku/renku.ini
```

`s3://` is used here but we should also support at least `nfs://`. Renku should
ask for credentials in a way that is appropriate for the type of resource that
is being added.

#### Getting information about the dataset:

```
renku dataset show very-large-data

Name: very-large-data
Created: 2022-03-30 15:55:59+02:00
Creator(s): Rok Roškar <rok.roskar@sdsc.ethz.ch>
Title: very-large-data
Description: Data that is very large. Indeed.
External Sources:
    - type: s3
      source: s3://server/path/to/bucket
      location: None
```

#### Staging external data:

External data has to somehow be brought into the system where Renku is running.
This seems to be a "staging" operation, but we could also call it something
else. "Mounting" is another option but seems too linux-specific.

```
renku dataset stage very-large-data
```

This command uses Fuse when necessary to mount data locally. Once it's staged,
it can be found locally and used in workflows:

```
renku dataset show very-large-data

Name: very-large-data
Created: 2022-03-30 15:55:59+02:00
Creator(s): Rok Roškar <rok.roskar@sdsc.ethz.ch>
Title: very-large-data
Description: Data that is very large. Indeed.
External Sources:
    - type: s3
      source: s3://server/path/to/<bucket>
      location: .renku/mounts/<bucket>
```

The data is accessible under `data/very-large-data/<bucket>` in the project's directory.

If the data is already mounted by some other means, `dataset stage` can be used
to let Renku know about it:

```
renku dataset stage --existing s3://server/path/to/<bucket>:/mounts/bucket
```

#### Streaming external data

The renku-python API should offer some support for more easily accessing remote
data, especially for sources from which it is common for data to be processed as
a stream. Ideally, Renku could lazily stage the external data if access to the
resource is requested.

## Drawbacks

* Relying on fuse is tricky because it requires elevated privileges on all
  systems. On linux, the user must have sudo and it isn't clear how this will
  work at all on windows. Fuse has the potentially to work very seamlessly but
  can also cause problems. *EDIT*: this turns out not to be an issue; outside of
  docker containers FUSE does not require additional privileges.

* It might also be inefficient to keep track of which data is actually external
  when recording/handling workflows.

## Out of Scope
* Saving credentials, so that RenkuLab users don't have to re-enter their credentials at the start of each session.

## Unresolved questions

* It's not obvious what happens to git-LFS. Do we eventually support pushing
  data back to e.g. S3 as well? This would actually not really be a drawback but
  maybe a plus?

* handling credentials on the deployment level becomes critical if workflow
  execution is off-loaded to a batch service*
