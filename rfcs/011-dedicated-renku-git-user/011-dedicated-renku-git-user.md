- Start Date: 16-12-2022
- Status: Proposed

# Dedicated Gitlab User for Knowledge Graph

## Summary

Every Renku deployment should have a dedicated regular user in every Gitlab it is connected
with. This user will be called something like `Renku` or `Renkubot` and it will be used
by the KG to access the project and process it's history.

## Motivation

The KG needs authenticated access to Gitlab so that it can analyze and index a project's 
Git history. Currently the KG has to store many Gitlab user tokens in postgres. 
These tokens are used depending on what project the KG is processing events for. This means
that the KG has to maintain and manage a database with Gitlab acces tokens (or project tokens).
If these are lost then the KG would not be able to access this project to add it in its database
during reprovisioning. 

In addition with the switch from Gitlab user access tokens to project tokens there was a
lot of automatically generated "bot" users that Gitlab created when the project tokens
were created. This is just what happens when a Gitlab project token is created but it may
be cause concerns with some admins or users.

Lastly, with having a regular user being dedicated to the KG, there is no more impersonation
of users and "fake" user activity that comes from the periodic checks and indexing of projects
that is done by the KG. All this activity will now be associated with the Renkubot user rather
than individual real users.

## Design Detail

### Helm Chart

In the Helm chart we would add a hook that would create a user in the Gitlab deployment
that Renku is connected to. The username and password will be saved in a secret. The KG
would then have access to the secret and use it to authenticate with Gitlab.

Alternatively the Helm chart can also accept existing user credentials. 

### Gateway

Either as part of this RFC or as an improvement the management of this secret / tokens for 
the Renkubot user should be done by the Gateway and the KG will simply either ask the Gateway 
for credentials or a Gateway's forward proxy will inject the right credentials in the right calls.

This will also be helpful when we support multiple Gitlabs/Github. In this case the KG does
not need to keep track which credentials to use for which Gitlab. The gateway can figure this
out and inject the right credentials for the Renkubot user.

### UI

There are some changes needed in the UI for this. Namely:
- To get a list of all projects for a user the UI would have to call Gitlab on behalf
of the user. This is because the KG will now only know about projects to which the Renkubot
user has been added. In other words, the KG will not know about any projects for which 
KG integration is not active. It could figure this out for public Gitlab projects
but it would be impossible to find private projects that are not in the KG.
- Creating a project on Renku will have to include adding this Renkubot user to the project
with `Reporter` permissions.
- KG integration for a specific project can still be verified with the KG. In some cases
the KG will simply not know anything at all for a project. In this case KG integration
for this project is not enabled. Checking that the Renkubot user is in the project with the
right permissions is not enough. It could be that the project has the Renkubot user but
the KG has not fully finished indexing the project.
- Enabling KG integration on projects that do not have it will have to be modified to also
add the user to the project similarly to creating a project.
- Disabling / removing KG integration results in removal of the project from the KG and removing
the Renkubot user from the project.

### KG

Depending on when the Gateway refactoring is completed, the KG may not have to deal with the Renkubot
credentials at all. This is the preferred option. But it may turn out that an initial implementation
of this gives the Renkubot credentials directly to the KG at first.

## Drawbacks

1. The Renkubot user will have read access to many repos. If the credentials are stolen or leaked
then they could be used to read the contents of private repos. 

2. When this is rolled out we would automatically add the Renkubot user to all projects. This may
be confusing to users.

3. Searching for projects and listing projects has to be done either solely through Gitlab or 
by going to both the KG and Gitlab.

## Rationale and Alternatives

The alternative to this is for the gateway to keep track of which user tokens can be used 
for which project and then provide these to the KG. This definitely complicates the design
and implementation of the gateway. If the gateway manages user access tokens for the KG we
would also have to make the database where the tokens are stored persisten. And losing the
tokens or letting them expire would mean that the KG would not be able to index projects
until at least one user for each project with the right permissions logs in.

An interesting benefit to this RFC is that we could get sort of a "naive" KG fedeeration with
the Renkubot users. For example if a KG for Renku deployment 1 gets the Renkubot credentials 
for Renku deployment 2, then Renku deployment 1 will be essentially able to generate the same
graph that is present in Renku deployment 2. Then, users of Renku deployment 1 will be able to 
search the KG of both deployments. But this is out of scope for this RFC - it is mentioned here
just as a potential benefit.

## Unresolved Questions

1. If a user is added to a project in Gitlab do they have to click a link in their email
to accept the invitation? If this is the case then this whole proposal is not feasible.

2. Do we implement simultaneously with the gateway refactoring or do we have an initial
implementation where the KG handles the Renkubot credentials.

3. Should we add the Renkubot users to existing projects when this is deployed for the 
first time.
