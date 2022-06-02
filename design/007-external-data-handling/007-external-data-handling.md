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
external sources. The primary concern to address is how to do properly track the
provenance of external data; Renku should preserve the information about the
original source of the data and provide mechanisms to retrieve this data when
necessary.

Note, however, that although this RFC focuses on the CLI, it should be assumed
that all the same applies to all other parts of Renku including Renkulab and
interactive sessions.

## Current status

### Git-LFS

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

### "External" dataset data

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

### S3

Using S3 in Renku is currently limited to interactive sessions. When launching a
session, users can specify any S3-compatible URL together with credentials, if
necessary. The S3 bucket is then mounted in their interactive session.
Currently, no metadata about these mounts is recorded. The data could, in
principle, be added to a renku dataset with the "external" mechanism described
above, but no metadata about its actual origins (i.e. an S3 bucket) would be
recorded.

### Upcoming

We have plans to include other external data providers, such as NFS shares,
openBIS etc. From the user's perspective, they all will generally work in the
same way as S3; data will be mounted inside the project and made accessible as
network-mounted storage.

## Related Work

The following section describes how similar tools handle external data.

### DVC - uses copying and file links

DVC supports [importing data](https://dvc.org/doc/use-cases/data-registry#data-import-workflow)
from external sources. When doing so, DVC saves dependency metadata about that data source.

DVC supports storing data in and retrieving data from a [remote
](https://dvc.org/doc/start/data-management#storing-and-sharing).
Data is copied between the local file system and the remote via explicit push
and pull commands.

If data is too big for the local file system, DVC supports using an [external
cache](https://dvc.org/doc/command-reference/add#example-transfer-to-an-external-cache).
The external cache works via file links.

### Snakemake - uses copying

Snakemake supports the use of remote files within pipelines.
When a file is needed for a pipeline execution, it is copied to the local file system.

From the [Snakemake documentation](https://snakemake.readthedocs.io/en/stable/snakefiles/remote_files.html):

> During rule execution, a remote file (or object) specified is downloaded to the
>  local cwd, within a sub-directory bearing the same name as the remote provider.
>  ... You can think of each local remote sub-directory as a local mirror of the
>   remote system.

### Kedro - uses programmatic access

Kedro's data access is based on configuration files ([Data Catalog](https://kedro.readthedocs.io/en/stable/data/data_catalog.html#the-data-catalog)).
In the configuration, kedro uses [fsspec](https://kedro.readthedocs.io/en/stable/data/data_catalog.html#specifying-the-location-of-the-dataset)
to provide a standardized interface to remote data systems.
Since kedro is python only, data is accessed programatically.
In other words, the units of kedro workflows are python functions that take as
input live python objects (dataframes, etc.), not file paths. fsspec helps map
a variety of remote data locations to live python objects.

## User stories

### 1. Sharing data for ad-hoc analysis

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


### 2. Tracking the use of external data

Sally is Bob's collaborator who continues to build on Bob's initial work by
building a more robust workflow. She uses standard Renku commands to record the
workflows. Renku uses the same mechanisms to attach the data as described above
when workflows are being re-executed or managed. She can also request to check
whether the results are still in sync with the data using `renku status`. In
this case, Renku checks the object hashes that were recorded previously against
the data currently found in the remote location (this might take a while over
the network - some data sources might provide pre-computed hashes). If the
hashes differ, Renku offers to update the results.

### 3. Data storage for research project

Willow is interested in a large, protected dataset that is offered for free to
researchers by an external organization. She has to accept a data access
agreement to access the data, at which points she receives credentials to the
bucket. To explore the dataset, she creates a Renku project and mounts the S3
bucket using the credentials.

After some exploration of the dataset, Willow decides to start a research
project with the dataset. She makes a copy of the dataset in her own private S3
bucket for two reasons: (1) to make sure she maintains access to the data
independent of the external organization, and (2) because she prefers to keep
the project’s raw, intermediate, and final data products in the same place. She
creates a Renku project for her work and mounts the bucket with her credentials.
Throughout the research project, she reads the raw data from the mounted
directory and saves intermediate data files and figures in separate folders in
the same S3 bucket.

### 4. Access remote data but make local copies

Sharing data in remote S3 buckets might not be only for large data that cannot
be stored locally; in some cases it might be prudent to make local copies. Users
frequently write their own supporting code for handling this. One such use-case
might look like this:

#### In the case where the data is small:

1. write a script to fetch the data and store it in a subfolder of the project
2. add the script to git
3. add the subfolder to gitignore

This way, the data is local and can be worked on in the context of the project.

#### In the case where the data is large:

1. write a script to fetch the data and store it on an external drive or another
   filesystem
2. add the script to git
3. write scripts that reference the external drive/filesystem and reduce the
   data to a manageable size
4. add those scripts to git
5. if the reduced data is very small, I will add it to git as well, if it is too
   large, I will put it in a gitignored subfolder

In this case, if the project is cloned to a new computer or the external
drive/filesystem is not available, the data can be fetched and reduced again
easily.

This approach leads to writing several one-off scripts and requires the user
to handle the data orchestration manually. Renku could offer commands to do away
with the boilerplate and ensure that data gets added/staged/mounted in a
repeatable manner for all users/collaborators of the project.

## Use-cases

From the user stories, two primary high-level use-cases emerge:

1. use data from remote storage as-is by remote-mounting it and remembering the
   source in renku metadata. This should support reading and writing.

2. use external storage as a source/sink but make local copies to speed up
   access for iterative calculations. In principle, this should also support
   reading and writing.

In the first case, we can do some book-keeping of data sources in renku metadata
and that’s about it. The performance might be bad depending on where the data is
with respect to the compute. Renku will need to recognize those paths when recording
and re-executing workflows as they are outside of git.

The second option is an extension of the first one. I.e. you define a data
source but then select certain parts from it to make local copies.

## Design Detail

Generically, we should strive to remove the separation between "hosted" Renku
(i.e. what I can do if I run an interactive session on renkulab.io) and
"offline" Renku (what I can do with the CLI anywhere).

Ideally we could once again follow the plug-in paradigm in the CLI, like we
do for other functionality.

### Provider interface and access modes

For each "provider", Renku should enable the user to copy data into a location
on disk as the default. For many use-cases, this would be sufficient. In cases
where large data is being used, additional options need to be considered.

Providers can leverage the well-supported
[fsspec](https://filesystem-spec.readthedocs.io/en/latest/index.html) or
[pyfilesystem](https://github.com/PyFilesystem/pyfilesystem2) libraries (though
see [this
discussion](https://filesystem-spec.readthedocs.io/en/latest/intro.html#other-similar-work)
contrasting the two). The interface should rely on a `DataSource` entity type
(or similar) that can be persisted in the knowledge graph. The role of the
Provider is to configure the project and/or the user's environment in such a way
as to make the `DataSource` available in a consistent, repeatable location with
respect to the project and without boilerplate.

Three scenarios are described here where a provider should ease the access to
external data _and_ work with Renku internals to record information about
provenance in the knowledge graph.

#### Mode 1 (default): copy

The default behavior of the CLI when staging the data locally (i.e. not on a
Renkulab instance) should be to copy the data into the project. When disk space
considerations allow it, this is preferred because data access will (almost)
always be faster than over the network.

#### Mode 2: mount

For large data that cannot be copied, providing a local mount is an option if we
adopt the view that data access should be done on the filesystem level. In this
case we must rely on
[FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace). `fsspec` (see
above) can use FUSE directly for many of the filesystem implementations. If a
new implementation is needed, the most sustainable way forward would likely be
to define an `fsspec` filesystem and FUSE mount it. It is not obvious that
relying on FUSE is a good idea at all -- we need to conduct a study of pros/cons
before committing to this path. There might be cases where FUSE would not be
universally acceptable - there, we should offer the option to use an
alternative, e.g. copy instead of stream the data.

#### Mode 3: reference

An alternative to using FUSE for large data would be to define data references
that would never be instantiated on disk, but only passed to Renku for recording
in the metadata. The user would be responsible for accessing the data with
whatever means necessary in their code; depending on the storage backend, Renku
could store checksums such that e.g. `renku status` could query the remote
source for any changes to the data. This approach wouldn't be too unlike the
current `--external` flag to `dataset add`, with the difference that there would
be nothing visible in the project's directory tree at all (using `--external`
makes a symlink). `fsspec` can be used here again to verify the metadata of the
resource with the storage backend. This is related to [renku-python
#2861](https://github.com/SwissDataScienceCenter/renku-python/issues/2861). When
working with references, another relevant library might be
[smart_open](https://github.com/RaRe-Technologies/smart_open) which provides an
interface for streaming large files from a variety of backends. In this case,
Renku could offer a utility (at least in the API) to provide access to remote
resources easily.

### Local vs. hosted interactive sessions

Relying on FUSE for providing filesystem access to external resources means that
locally we do not require special privileges and can manage the data access for
the user, provided they are able to install and use FUSE on their system. In the
docker-based remote sessions, the situation is more complicated because
containers require elevated privileges to access FUSE on the host. Therefore we
will need to expose a microservice that will provide management of external data
sources for the user. The renku CLI would be aware that it is running in a
hosted session and instead of executing a local command for mounting the data
source, it would send off an API request to the service. The service itself
could run in a sidecar or be integrated with datashim, which is currently used
for providing S3 buckets in interactive sessions. If done via a sidecar, we
could likely achive dynamic mounting of data sources, which would be fantastic
for data exploration but comes with security trade-offs (sidecar needs to run
with elevated privileges). Note that for NFS mounts, _root is still required_. A
fuse-nfs project exists, but elevated privileges are needed for non-root users
in order to use it.

Because of the security considerations, dynamic mounting needs to be a feature
that can be turned off in more restrictive environments. The alternative that
will be provided is to mount volumes at the start of the user session, requiring
a restart if a new volume is needed. That way, an entirely separate service can
handle the volume provisioining, as is the case now with the current
implementation of S3 buckets.


### Commands

Below are some examples of potential usage of external data with existing and
new `renku` commands.

There are a few situations to consider. In all cases the data referenced from
external storage should be trackable through renku workflows and the Renku API
should offer utilities to access that data.

1. External (raw) data exists in some external location and it would be useful
   to bring it to Renku and annotate it in the context of a Renku dataset. The
   difference with current functionality is that the user does not want to add
   this data to git-LFS.

2. A Renku dataset is created but instead of using git-LFS it should be backed
   by another external data store (perhaps for easier access from other
   systems). This means that _local_ data will be added to the dataset and
   (eventually) sent to the remote storage.

3. External data is brought into the project but _not_ in the context of a
   dataset (that could be done at a later point). Renku should remember that
   this data has been brought into the project so that other users of the
   project can also easily stage it when needed. As above, the data should _not_
   be pushed to git-LFS but instead kept on the remote resource.

In all three of these cases, at least two of the access modes described in the
previous section should be accommodated, namely "copy" and "mount". In the case
of datasets, it's not clear how "reference" is to be used; however, it doesn't
seem contradictory to organize data references in a dataset and make them
accessible via Renku's `Dataset` API.

Note: in the commands below `s3://` is used as an example but all of the support
libraries referred to above support common APIs for (nearly) all likely storage
backends and automatically switch between them depending on the protocol of the
URL. That's a long way of saying that one can replace `s3://` with `hdfs://`,
`nfs://` or `gcs://` etc.

#### Creating a Renku dataset backed by external storage

```
renku dataset create s3-backed-dataset --storage s3://server/path/to/bucket
```

This command creates a dataset and sets its backing storage to be S3 instead of
the default git-LFS. Locally, the directory `data/s3-backed-dataset` is created
and added to `.gitignore`.

```
renku dataset add s3-backed-dataset /path/to/file/on/filesystem
```

This adds a file from the local filesystem to the dataset. It is _copied_ to
`data/s3-backed-dataset`. (Should it also be pushed to the remote storage
immediately?)

#### Adding external data to a Renku Dataset

This situation is different than the previous one in that remote data already
exists but the user wants to bring it into a Renku context. The usual `renku
dataset add` is used but Renku interprets the `s3://` URL and automatically
registers it as an external dataset:

```
renku dataset add --create external-dataset s3://server/path/to/bucket
-- prompt --
Access Key: 1234
Secret Key: abcd
Store credentials? (y/N): y
-- output --
Credentials stored in ~/.renku/renku.ini
```

On the filesystem, the command creates `data/external-dataset` as is normal for Renku
datasets. Importantly, `data/external-dataset` is also added to `.gitignore` to avoid
exploding the git repository. 💥

Note that by adding the data, only the metadata objects are created, the data is
not actually staged yet.

#### Getting information about the dataset:

```
renku dataset show external-dataset
-- output --
Name: external-dataset
Created: 2022-03-30 15:55:59+02:00
Creator(s): Rok Roškar <rok.roskar@sdsc.ethz.ch>
Title: external-dataset
Description: Data that is very large. Indeed.
Data Sources:
    - type: s3
      url: s3://server/path/to/bucket
      location: None
```

Note that `location: None` because the dataset has not been staged yet. See
below for details on that. It is possible for a dataset to contain data from
multiple sources. In the case of new data added to the dataset locally, the user
should be able to decide which external location to push the data to either at
push time or when adding the data files to the dataset.

#### Staging external data:

External data has to somehow be brought into the system where Renku is running.
This seems to be a "staging" operation, but we could also call it something
else. "Mounting" is another option but seems too linux-specific.

```
renku dataset stage external-dataset
```

This command copies by default. If the user wants to avoid copying and stage by
mounting, FUSE can be used. For some filesystems we might be able to warn the
user that the data they are trying to stage is very big and they should use the
`--mount` flag instead.

```
renku dataset stage --mount external-dataset
```

The default location is `data/<dataset-name>` but can be specified with the
`--location` flag in case it is preferred to take the data to an external drive
or another filesystem - in that case, the data is linked to
`data/<dataset-name>`. For example:

```
renku dataset stage --location /scratch/ external-dataset
```

would result in `/scratch/external-dataset` --> `data/external-dataset`.

Once it's staged, it can be found locally and used in workflows:

```
renku dataset show external-dataset

Name: external-dataset
Created: 2022-03-30 15:55:59+02:00
Creator(s): Rok Roškar <rok.roskar@sdsc.ethz.ch>
Title: external-dataset
Description: Data that is very large. Indeed.
External Sources:
    - type: s3
      url: s3://server/path/to/<bucket>
      location: /scratch/external-dataset
```

The data is accessible under `data/external-dataset/<bucket>` in the project's directory.

If the data is already mounted by some other means, `dataset stage` can be used
to let Renku know about it:

```
renku dataset stage --existing /mounts/bucket s3://server/path/to/<bucket>
```

#### Using external data _without_ Renku datasets

We should also consider the case where creating a dataset is too much effort or
does not fit the stage of the project. In such cases, Renku should still allow
the user to bring in and reference external data. For this, we should review the
existing `storage` subcommand. For example, to bring in data from an S3 bucket,
the user would do

```
renku storage add s3-bucket s3://server/path/to/<bucket>
-- output --
s3-bucket added as external storage
```

This would not immediately do anything with the data, but only create the
`DataStorage` metadata object. For bringing in the data, the user would need
to do

```
renku storage stage s3-bucket path/to/bucket
```

The `--copy/--mount` semantics are the same as in the `dataset` commands
above. The `path/to/bucket` is entered in `.gitignore`. If the `--mount` flag
is used, the data is mounted instead of copied. Same as above, the `--location`
flag can also be used to specify a different location for staging the data,
which is then linked to `path/to/bucket`.

The above can be condensed in

```
renku storage add s3-bucket --to path/to/bucket s3://server/path/to/<bucket>
```

This is stored in the metadata so when the project is cloned elsewhere, the
relative location of the data is fixed. So, e.g.

```
renku storage ls
-- output --
Type |    Name   |              URL              |        Location        |  Project path
--------------------------------------------------------------------------------------------
S3   | s3-bucket |  s3://server/path/to/<bucket> | .renku/mounts/<bucket> | path/to/<bucket>
```

To quickly get access to all of the external data when cloning the project
in a new location, the user can easily copy/mount all the known remote data sources:

```
renku storage stage --all
```

Note that in the case of remote storage (as opposed to datasets) there is no
contradiction in writing directly to the directory where the data source is made
available in the local project. If the data source is mounted, then the changes
are reflected on the remote location immediately; if the data is copied, it
should be possible to implement a `sync` command that would copy upstream new or
changed files.


#### Syncing changes with remote storage

In the default "copy" mode, there must be a way of syncing data back to the
remote storage (and to potentially update local copies with new content).

```
renku storage sync
```

Renku should detect conflicts (when both the remote and local have been updated)
and give the user the option to either persist/keep the local or remote copy.
Alternatively, it should let the user abort the action altogether to avoid
accidental data loss.

#### Streaming external data

The renku-python API should offer some support for more easily accessing remote
data, especially for sources from which it is common for data to be processed as
a stream. Ideally, Renku could lazily stage the external data if access to the
resource is requested.
[smart_open](https://github.com/RaRe-Technologies/smart_open) might be useful
for this.

## Drawbacks

* Relying on FUSE is tricky because it requires elevated privileges on all
  systems. On Linux, access to `/dev/fuse` and `CAP_SYS_ADMIN` in Docker is
  needed for fuse to work. NFS mounts usually require root privileges.

* In order for the mounts in interactive sessions to be dynamic, the sidecar
  container needs to use bi-directional mount propagation, which requires
  privileged mode. This solution might be unacceptable in some cases so we might
  need to consider dynamic mounting to be a feature that can be disabled. On the
  other hand, user namespaces seem to be [coming to Kubernetes
  "soon"](https://github.com/kubernetes/enhancements/pull/3275) in which case
  privilege escalation in containers will be less menacing.

* Unclear how to keep track of which data is actually external when
  recording/handling workflows.

## Out of Scope
* Saving credentials, so that RenkuLab users don't have to re-enter their
  credentials at the start of each session.

## Unresolved questions

* It's not obvious what happens to git-LFS. Do we eventually support pushing
  data back to e.g. S3 as well? This would actually not really be a drawback but
  maybe a plus?

* handling credentials on the deployment level becomes critical if workflow
  execution is off-loaded to a batch service

* Syncing data to a remote location sounds an awful lot like needing to
  implement version control on top of data files - there might be some
  alternatives to this already.

* When syncing data with the remote location, how should potential conflicts be
  resolved? On shared infrastructures it's usually not the infrastructure
  provider's role to ensure that users aren't overwriting each other's data if
  using a shared filesystem, but maybe we need to at least try to have some
  sanity checks?

* On `renku dataset stage` should Renku check the checksums of the staged data
  and compare them with local checksums? This seems especially pertinent if
  Renku is pointed at an existing location for a dataset.

* It would be nice to have some filtering on the remote data - should that be in
  scope here?
