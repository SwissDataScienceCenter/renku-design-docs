# Let users pin projects to the Dashboard

Authors: Flora Thiebaut, Laura Kinkead

## 🤔 Problem

As a user, I have a handful of projects (i.e. up to 5) which I want to always have a quick link to. Currently, if I explore projects on RenkuLab, my own projects may disappear from the Dashboard because of the “last-visited-shown-first” logic.

## 🚞 User stories / journeys

- As a user of RenkuLab, I want to be able to pin a project to the Dashboard. (Most likely from the header card?)
- As a user of RenkuLab, I want to un-pin a project from the Dashboard.
- As a user of RenkuLab, I want to re-order my pinned projects. (Is this a nice-to-have?)

## 🍴 Appetite

3 weeks

## 🎯 Solution

This is very similar to GitHub’s pinned repositories

### User Flow: Pinning and Unpinning

User should be able to pin a project from its entry in the Dashboard and from its project page.

From the same places, the user should be able to un-pin the project.

### Re: Indexing

Pinned projects should show up, even if the project is not indexed in the KG.

### Nice-to-have: Custom ordering

From the Dashboard, the user can reorder pinned projects. Maybe via a simple menu, or by drag and drop.

### How Pinning & Recently Viewed play nice together

Note that there should be some maximum number of pinned projects (5), so that the Dashboard can still display at least 5 last-visited projects (Total is 10 projects = X pinned + Y last visited).

Show pinned projects and then below show recently viewed.

Projects with running sessions: if the project is pinned, it stays exactly where it is. If the project is unpinned, it goes to the top of the unpinned section (below pins).

[nice-to-have] In the same “Customize Dashboard” menu where the user chooses the ordering of their the pins, allow the user to toggle on or off also showing recently viewed projects.

### Technical Details

Add some backend to track which projects are pinned by a user and in which order, i.e. add a new user preferences service, or add this feature to it.

## 🐰 Rabbit Holes

- Potentially the first time the UI team will add a backend service in Python

## 🙅‍♀️ No-gos

- Pinning Datasets
- Showing pinned projects in search
- Pins should not be saved in the KG
