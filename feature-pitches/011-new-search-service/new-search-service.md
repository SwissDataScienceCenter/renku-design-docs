# Renku 2.0 Project and User Search Service

Authors: Laura Kinkead

> [!IMPORTANT]
> We are building the next version of Renku! (Currently referred to Renku 2.0) Would
> you like to get involved in shaping the future of Renku? Interested to participate in our user
> research? Get in touch! hello@renku.io

## 🤔 Problem

In Renku 2.0, we want projects, users, environments, etc. to be searchable. This pitch is the first
start towards the new Renku Search and Discovery services.  

## 🍴 Appetite

6 weeks

## 🎯 Solution

RenkuLab 1.0 should have a search page. The search functionality should fulfill the following user
stories.

### 🚞 User stories / journeys

#### Search by keyword

When I am looking for projects by a keyword, I want to view results of projects where that keyword
is present in the [title, description], so I can find relevant projects easily.

#### Search ordering

When I search the name of a project that I remember, I want that project to be the first search
result, so that I don’t have to scroll to find exactly what I know I’m looking for.

#### Fuzzy matching

When I am searching for projects by keyword, I want to see results with fuzzy-match so that I don’t
have to do multiple searches to find projects relevant to my topic.

(ex: searching ‘bioinf’ returns hits for ‘bioinformatics’)

#### Performance

When I am searching RenkuLab, I want to view search results within [3?] seconds, so that I am not
annoyed at how long it takes.

#### User search

When I find an interesting project, I want to be able to click on the project author’s name and view
all of their (public + internal) owned projects [in a pre-populated search page] so I can explore
what other projects from this person might be relevant to me.

#### Search filtering: by me

When I click the “View All My Projects” button on the dashboard, I want to see projects that are
created by me or owned by me on the search page, I want to be able to narrow my search via the
search box with my “owned by me” filter already in place, so that I can find my own projects
quickly.

#### Search filtering: user filtering [nice-to-have!]

When I remember a project that was created by another person, but don’t quite remember the name of
the project, I want to be able to search for project created by a specific user, so I can find the
project.

This might be accomplished via a UI filtering element, or via a DSL like `metrics by:Rok`.

## 🐰 Rabbit Holes

## 🙅‍♀️ No-gos

Datasets will not be searchable in this first iteration.
