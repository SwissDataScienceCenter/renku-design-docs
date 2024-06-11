- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Status: (One of Proposed, Accepted or Rejected)

# Use devcontainer spec for Renku projects

## Summary

The [development container](https://containers.dev) concept grew out of a need for easily encapsulating
runtime environments for developing software projects. It has now been turned into a spec with an ecosystem
of tools growing around it; a project using a devcontainer definition can get a docker container provisioned
locally by VSCode, remotely by codespaces, or even more remotely in any cloud using [devpod](https://devpod.sh).
Devcontainers envision composing a runtime environment flexibly from basic components, minimizing the burden
on the user/developer who just wants to get work done and doesn't want to worry about writing Dockerfiles or figuring
out how to install the latest cuda libraries.

The proposal here is to use the devcontainer concept for Renku projects.

## Problem Statement

Renku projects are currently bootstrapped via a template which creates a number of files that define the runtime
for that project, e.g. the `Dockerfile` and depending on the languages used, some other supporting enviroment files.
In order to be able to run images for Renku projects on the hosted infrastructure, we provide some base images that
we know will work when used on e.g. renkulab.io. This has a number of issues:

1. if users wants to modify just about anything other than installing python packages, they have to know how to write a Dockerfile (not easy)
2. we need to maintain a library of docker images for a variety of project dependencies and scenarios (a huge pita)
3. enabling new functionality is very difficult because it requires developing new docker images or adding things to existing ones

## Key Assumptions

We assume that we want to continue working in containers...

## Possible Solutions

Change nothing, it works!

## Proposed solution

* Create a devcontainer renku feature (see [here](https://github.com/rokroskar/renku-devcontainer-feature/))
* Use `devcontainer.json` in renku projects (see [here](https://dev.renku.ch/projects/rokroskar/devcontainer-test))

The end-result from the user's perspective is that there is no more Dockerfile; the only configuration left is something like this:

```yaml
{
    "name": "Python 3.10",
    "build": {
        "context": "..",
        "dockerfile": "../Dockerfile"
    },
    "features": {
        "ghcr.io/rokroskar/renku-devcontainer-feature/renku:1": {},
    },
    "forwardPorts": [8888],
    "remoteUser": "jovyan"
}
```

(Ok there is a `Dockerfile` but it contains only a `FROM` line and nothing else).

If I now, for example, wanted to add cuda to this project, all I need to do is add the feature:

```yaml
{
    "name": "Python 3.10",
    "build": {
    "context": "..",
    "dockerfile": "../Dockerfile"
    },
    "features": {
        "ghcr.io/rokroskar/renku-devcontainer-feature/renku:1": {},
        "ghcr.io/devcontainers/features/nvidia-cuda:1": {}
    },
    "forwardPorts": [8888],
    "remoteUser": "jovyan"
}

```

The gitlab-ci looks like this:

```yaml
...
  script: |
    CI_COMMIT_SHA_7=$(echo $CI_COMMIT_SHA | cut -c1-7)
    devcontainer build --workspace-folder . --push true --image-name ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA_7}
```

The user can now launch this project on any RenkuLab instance as before, _and_ in their local VSCode they can spin it up with the built-in devcontainer tooling.
This means that we essentially no longer have to worry about local sessions.

The extra bonus is that with a tool like `devpod`, users can launch their projects on any other cloud, with no changes required to the project itself.

An additional benefit is that we can have some control over which versions of the Renku CLI are installed by controlling the release of the renku feature rather than
having to modify/update users' projects.

If we decide to use devcontainers for projects, we can think about deeper changes for using this ecosystem, e.g. replacing parts of the notebook service / amalthea or
at least making them devcontainer-aware.

## Drawbacks

One drawback is that this cannot only be done in templates but will require changes to the UI as well so the blast radius of this change isn't very small.

Another problem is that until we have some deeper support for the spec in our services, some features of the devcontainer will not be used, notably the lifecycle hooks.

## Rationale and Alternatives

This would make Renku projects much more portable and interoperable while at the same time allowing us to _completely stop supporting an entire library of docker images_.

## Unresolved questions

* What to do about data sources / mounts? We currently do some custom things for data source management and it's not obvious how that can translate into this context.
