- Start Date: 2022-06-24
- Status: Proposed

# Improved handling of Git forks of Renku projects

## Summary

Add a process of detecting and handling forks for projects and their child metadata classes.

## Motivation

There are many cases where forking a project leads to duplicate metadata and unclear ownership of metadata, for instance a dataset that is modified and belongs to two different projects, showing up as two results in our search with the same name and description. This leads to performance hits due to the amount of data that needs to be searched, makes it difficult to deduplicate the results and only show relevant entries as well as leading to hard to solve situations where different projects depend on different sources of the same entity.

We need a better process of handling these situations, to keep our metadata clean and clearly differentiate between uses and ownership of metadata.

Example cases that currently cause headaches:

- Modification of the same dataset in different forks
- Modification of the same Plan in different forks
- Modifications of imported datasets

## Design Detail

Forks in Git have a use-case that is perfectly valid, namely if a user wants to make a modification to a project he doesn't have access to, and they want to push those changes to the upstream project. This solution should continue supporting that use-case.

The main approach taken here is that we differentiate between Git forks and Renku forks(needs a better naming).

### Git fork

Done as usual through git/gitlab, just normal forks. Projects forked this way are NOT picked up in our metadata, other than a dummy project node in the KG. So they don't have the Datasets or any other metadata of the source project show up in the KG. They are purely intended for creating upstream PRs or playing around.

### Renku fork

A Renku fork is created through the core service or running a command, e.g. `renku project fork`. It gets a new node denoting it as a fork and the user has to specify a new name, and the fork node in the metadata points to the original project with a `schema:isBasedOn` edge.
All datasets of the original project do NOT belong to the new forked project node and as such do not matter when generating triples for the forked project.

A user cannot modify datasets or plans from the original project in the forked project, though they can execute the plans to create activities. They also don't show up through the hasDataset or hasPlan edges on the forked project.

If a user wants to modify a Dataset or Plan from the original project, they have to run a command like `renku dataset fork` or `renku workflow fork`, supplying a new name and possibly description, which creates a new dataset or plan with a `schema:isBasedOn` edge pointing to the original one.

Imported datasets follow the same logic and also need to be forked before they can be edited.

#### Turning a Git fork into a Renku fork

A user can turn a Git fork into a Renku fork using a command like `renku project fork`. They need to specify the remote of the original project so we can use `git merge-base --fork-point` to determine the common ancestor. If the user didn't modify any datasets or plans from the original project, this operation can just go ahead and create a new Project node with a `schema:isBasedOn` edge as mentioned in the previous section. Datasets that were created after the fork can just be moved over.

If a user did modify a preexisting dataset before running this command, they are prompted to provide a new name for it and it's provenance chain gets split by changing the `wasDerivedFrom` edge at the fork point to a `schema:isBasedOn` edge.

### Migration

Existing forks get treated as Git forks and their users have to turn them into Renku forks with `renku project fork`.

## Drawbacks

Existing forks would disappear until their users run `renku project fork`.

Projects that previously imported a Dataset from a fork might see the upstream disappear and would need some manual intervention to fix or depend on the upstream turning their Git fork to a Renku fork.

Might be a bit unintuitive for users to have two different concepts of forking, we definitely need good naming for it.

## Rationale and Alternatives

This approach clearly separates plain git forks that are done as part of the normal developer workflow from forks that are meaningful for Renku metadata. This way, it is up to a user to decide whether they think their changes are relevant.

The alternative is business as usual, try to deduplicate datasets/plans in
queries, which is clearly untenable as there is no sure-fire way to pick which
dataset is the correct one and what is relevant to a user.

We cannot auto-detect forks, so any approach based on that wouldn't work, and modifying triples for forks in graph generation also wouldn't work, as there would still be many versions of the same e.g. Dataset and we can't just assign new meaningful names to them automatically.
Similarily, just having a `renku dataset fork` command that allows editing imported datasets wouldn't solve our problems, as git forks could still result in modified datasets that are the same as original datasets.
## Unresolved questions

