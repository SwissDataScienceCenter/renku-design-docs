# Set custom culling times per resource pool

Authors: Laura Kinkead, Rok Roškar

## 🤔 Problem

Users often don’t shut down their sessions when they are done. This leads to cluster resources being used up unnecessarily, and subsequent users not being able to start sessions.

This problem is especially pressing in the context of courses, when many students start sessions at the same time and use a lot of cluster resources concurrently.

Fortunately, this also presents an opportunity. Whereas a researcher might come back to a session for multiple days in a row, a student probably only needs their session for the course day.

Currently, we cull sessions (hibernate the session) when they have been inactive for 12 hours.

## 🍴 Appetite

1 week

## 🎯 Solution

We should be able to set custom session culling times for resource classes, and therefore set shorter culling times for course participants or for expensive resource classes with GPUs or large-memory sessions.

Culling time is defined as the amount of time that a session can sit idle before we automatically hibernate the session.

The culling time for each resource pool should be configurable in the admin panel.

The user should be made aware of the culling time of their session.

### 🚞 User stories / journeys

[admin - must have]

When I add or modify a resource class in the admin panel

I am given the option to set a custom culling time

so that sessions using that class don’t continue running for too long

[user - nice to have]

When I start a session and select a resource pool,

I see what the culling time is for that resource pool,

so I know how long I can leave my session paused before it will be shut down.

## 🐰 Rabbit Holes

- Changing the culling time on an existing class does not impact currently-running or hibernating sessions.

## 🙅‍♀️ No-gos
none