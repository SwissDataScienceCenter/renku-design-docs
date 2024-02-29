# Renku 1.0 Project Page UX

Authors: Laura Kinkead

> [!IMPORTANT]
> We are building the next version of Renku! (Currently referred to Renku 1.0) Would
> you like to get involved in shaping the future of Renku? Interested to participate in our user
> research? Get in touch! hello@renku.io

## 🤔 Problem

Renku 1.0 needs a whole new Project Page. The components of a project in Renku 1.0 are fundamentally
different than current projects. For example, Renku 1.0 projects may have multiple sessions, and
therefore need more than 1 start button. Renku 1.0 projects may have more than 1 code repository, so
having 1 files tab doesn’t necessarily make sense anymore. This pitch is **not**  “let’s improve the
UX of the current project page”. It is a fundamental rethinking of what the project page is.

There are a few ways in which the new Project Page needs to operate differently:

- Need to show multiple code repositories connected to a project
- Need to show multiple sessions
- Want to show project storage as a component with as much importance as datasets and sessions.

In addition, we have some new overall UX goals:

- We want the project creation process to be much simpler, so users are not overwhelmed. We often
  hear users say that they don’t want to use Renku because “Renku is too much”, and “I don’t need
  all of this”. Starting a new Renku Project should feel lightweight.

## 🍴 Appetite

6 weeks

## 🎯 Solution

> [!IMPORTANT]
> This pitch will be design-only. Implementing the design will come in a later build cycle.

### Simplifying Project Creation

A special focus for designing the new Project Page is to simplify creating a new project. We want
users to feel that:

- Creating a new project is quick, easy, and lightweight
- It’s obvious what to do next when you create a project
- You don’t have to use *all* of the components of RenkuLab in your project

I could imagine that some things that would help achieve this goal would be to have some suggestions
on the fresh project page. For example, not just an option to “Add a Repo”, but a “Add a GitHub
Repo” shortcut button. And not just an option to “Add a session”, but a “Add a Session from the
Python Data Science template” shortcut button.

### Project Page Elements

A Project has the following properties:

- Metadata
  - Name
  - Creator
  - Short description
  - Keywords
  - Avatar (image)
  - Visibility (public, private)
  - Members
    - Access rights
- Components
  - Code Repo(s)
  - Session(s)
    - Default Compute configuration/resource pool
    - Running sessions
  - Dataset(s)
  - Storage(s)
    - Default storage configuration
  - Workflow(s) ?
- Other
  - Long-form description
  - Publication(s) ?
  - Share button
  - Adding labels? or categories ?
- Settings & Admin
  - Update status
  - Indexing status ?

## 🐰 Rabbit Holes

### Target Audience: Project Creators

The target audience for the project page is the Renku user who is **developing the project**.
Project pages also function as a place to share and showcase work, and are used by external project
*viewers.* However, that user group represents a secondary goal, and are not the focus of this first
build.

💭 Note: We have had some discussion so far about in the future implementing a “project publication”
concept. Once a user wants to share their project with others, they would flip some switch to
indicate that the project is ‘published’/‘polished’/’curated’, and this alters how the project page
looks to cater to an external audience rather than the project developer. Published projects would
also be prioritized in search results before un-published projects.

## 🙅‍♀️ No-gos

- Changing the main top navigation bar and the footer is not in scope for this pitch
