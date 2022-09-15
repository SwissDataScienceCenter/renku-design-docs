# Broadly Accessible Workflows via a Workflow File

This feature will make Renku workflows more functional for more experienced coders. But even better, it will make workflows easier to teach and learn, and make the world of workflows accessible to a broader audience.

## 🤔 Problem

Workflows are Renku's reproducibility “last frontier”. Renku helps users with code versioning, dataset metadata, environment containerization... but workflows remain one of Renku’s least used features. Why is this?

CLI tools require the user to maintain a mental model of what’s happening with every command they run. This additional layer of abstraction is the reason why we only expect advanced users to use CLIs at all. For example, a person who understands git well gets the job done quickly with CLI commands, but someone new to git will prefer to use their IDE’s GUI or the GitHub web or desktop app.

As long as Renku workflows are only available on the CLI, only the most advanced users will use them. But Renku’s mission is to make reproducible research *easy* and for *everyone*, not only the computationally advanced! So, our first motivating question is *what can we do to make workflows accessible to a broader audience?*

Additionally, there is a common challenge we run into when introducing Renku workflows to users who *do* want to use workflows: it is cumbersome/just-not-tenable to create and develop (modify) workflows of more than a few steps. Many workflow users have 10+ steps in their workflows [[source](https://sdsc.atlassian.net/wiki/spaces/RENKU/pages/2246541341/Reproducibility+on+HPC)]. Entering 10+ steps on the command line is tedious to do once, but even more significant is that modifying that workflow- which users do all the time as they develop their pipeline- requires recreating and rewiring steps again and again on the CLI. Some users end up writing a script to call a series of Renku run commands [[ORDES example](https://renkulab.io/projects/renku-stories/digirhythm/files/blob/src/build_workflow.sh)]. Renku should provide a built-in way to create and develop multi-step workflows via a file!

## 🍴 Appetite

We’d like to invest 2 sprints (6 weeks) for making workflows more accessible to a broader audience.

## 🎯 Solution

We’d like to offer the ability to define Renku workflows in a file, where the full workflow is laid out for the user to plainly see and edit.

I’ve done a [survey of popular workflow tools](https://sdsc.atlassian.net/wiki/spaces/RENKU/pages/2310701105/Workflow+Tool+Research), and YAML seems to be a common choice for a workflow file, but we leave the final decision to the team.

<img src="fat-marker-file.png" width="200">

The workflow file names a sequence of steps, and each step names its command, inputs, outputs, parameters, etc. Each step should be named so that it can be referenced for running individually. I imagine that the structure of the workflow step definition in the file would be pretty similar to the output of `renku workflow show`.

![renku workflow show](workflow-show.png)

For the scope of this pitch, expressing `renku workflow iterate` in the workflow file definition syntax is _out of scope_, along with any kinds of looping or branching.

### Editing workflow files

The workflow file is opened and modified just like any other code file in the project. This pitch requires no changes to the UI.

The user should be able to create more than one workflow file in a project.

### Running workflow files

The user would still use the CLI to run the workflow, referencing the workflow via the workflow file name. To improve usability for non-CLI users, in tutorials, we could perhaps demonstrate running shell commands from inside a notebook (`!renku workflow execute workflow.yml`).

Developing a workflow should be as simple to the user as editing the workflow file, executing the workflow file on the CLI, making further modifications to the workflow file, executing the workflow file again, etc. Renku should not interrupt or slow down this development process. 

The user should be able to run a specific named workflow step from a file without running the whole file (a la `renku workflow execute workflow.yml step_1`, or even `renku workflow execute step_1` if we can provide searching all workflow files). As a _nice-to-have_, it would also be nice to support running a subset of steps, for example `renku workflow execute workflow.yml --from step_name_3 --to step_name_6`.

### Improving workflow user awareness: A workflow file template

It is worth considering adding a template workflow file to the default Renku project structure. For example, this could be a workflow file with an example workflow step written out but commented out. The user can uncomment it and replace the placeholders with their command, inputs, output, etc. There could also be a comment at the top of the file describing how to run the workflow.

This would put workflows in front of more users, increasing awareness and hopefully also adoption. 

### Note: Re-runnability & Pre-existing Outputs

A common frustration users have with current Renku workflows is that the `workflow` & `run` (mostly `run`) commands errors when you run a workflow command that writes an output file that already exists.

Users do this frequently during development: run script `a`, generate file `b`. Change script `a`, rerun, generate a new b. Great, now you're happy with script `a`, so `renku run script_a` one last time to record it. But now Renku complains that no outputs were generated (because file `b` already existed, so nothing was detected)! Users *can* avoid this by telling Renku explicitly that the output is file `b`, but that's more typing. More commonly, users end up deleting output files before every `renku run`, which they- very reasonably- don't like doing.

The file-based workflow feature should solve this. The workflow should read the specified output in the workflow file, rather than detecting changed files. This removes the need for the user to delete the file so that Renku can detect its creation.


## 🐰 Rabbit Holes

### Renku Workflow Metadata

One of the characteristics that prevents users from using Renku workflows and `renku run` is that it feels very *final*. Renku records what you do, and there's no taking it back. Workflow structures are immutable, so when your workflow changes, you have to create a new one.

This approach maintains a consistent metadata model, but it is not tenable for users! Imagine having a 10 step workflow, and then you modify the number of inputs that the script in step 4 takes. You have to create a new step 4, and then create a new `renku workflow compose` to put the 10 steps back together again.

File-based workflow definitions help with this by providing an editable file- just modify step 4 and re-execute the workflow file- all done. But what does this mean for workflow metadata?

**What happens when you modify a workflow file?**

It makes sense to maintain that Renku workflow metadata are immutable. Also, for the vast majority of users, there is no cost to having lots of workflow metadata objects in the background. So, I suggest that editing and running modified workflow file should generate a new workflow metadata object (`plan` and `activity`).

The consequence of this is that there is no connection between iterations of a single workflow file as the user develops it. We recognize that this not *ideal* (for example, the user will only be able to compare `activities` generated by a non-modified `plan`; once they change the workflow, activities are not easily comparable anymore), however we consider it a worthwhile tradeoff for the sake of making workflow development an easy user experience.


**How do we meaningfully list *relevant* workflow metadata with so many pieces of workflow metadata floating around?**

The one cost of creating lots of workflow metadata is that listing the project's workflows yields a lot of metadata that the user doesn't care about. But the solution to that is right in the statement- the user doesn't care about any metadata that relate to data objects that no longer exist. `renku workflow ls` should only list workflows that relate to currently existing code and data objects, i.e. show me the workflow that I ran to create the current version of file `b`, not the workflow I ran yesterday that created a previous version of file `b`. (In the `plan` and `activity` vocabulary, only `plans` that have an `activity` that relates to a file currently in the project)


### Misc Rabbit Holes

**How do you handle when a workflow step generates many files?**

As is the state of the current workflow functionality, inputs and outputs may be directories, rather than files, to support multiple files. However we don’t support any kind of wildcard syntax.

**How does this feature mesh with the current workflow parameter file functionality?**

Currently, the user has the option of creating a file specifying workflow parameters, and passing this in to `renku workflow execute` to override the default parameters. In the future, the combination of this parameter file and the workflow definition file could develop into a sort of base workflow + several “variation” files defining a set of related workflow experiments. But work into this direction is _out of scope_ of this pitch. 


**Should one workflow file be able to reference another workflow file? (i.e. to compose workflows between files)**

My default is *no*, keep it simple for the scope of this pitch.

**Potential for Un-executable `Plans`**

In the current `renku run` implementation, a `plan` is only created after the command executed successfully. By importing and creating the `plan` upon executing the workflow file, the user could potentially end up with a non-executable `plan` in their project. This sounds non-ideal, however, it doesn’t really have any implications for the user, so it’s up to the team for how they would like to handle this behavior.

## 🙅‍♀️ No-gos

There are a lot of feature-rich workflow tools out there already. We’re not trying to make the *best* workflow tool, and not even a comprehensive one. If we can get a researcher started using workflows for the first time because simple workflows are easy to create in Renku, and then that user switches to a different workflow tool that suits them better, that’s a win for Open Research!

We’re not trying to make a comprehensive workflow tool with lots of functionality. We’re trying to make the *minimal* workflow tool a user can learn, so they can take their research project to the next level of reproducibility. So, the file-based workflow syntax need only support a minimal set of workflow functionality.

Specifically, expressing `renku workflow iterate` in the workflow file definition syntax is out of scope, along with any kinds of looping, branching, and wildcard syntax.
