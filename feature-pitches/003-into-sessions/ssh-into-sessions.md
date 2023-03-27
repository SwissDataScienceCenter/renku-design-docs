# Support RenkuLab compute access from local terminal (SSH) & VSCode

Authors: Laura Kinkead
Build Dates: January 16, 2023 → March 10, 2023
Created by: Laura Kinkead
Opportunity: https://www.notion.so/I-want-to-work-on-my-own-machine-f0a9b3baca7743bc897c13fdf4b30a1c
Priority: 🔴 Must Have
Related Tech Topics: https://www.notion.so/2c27d2b88bf74d98a3f582cdf2cd9ec8, https://www.notion.so/7a48e271195c4255b32b204994167cc0
Size: 🌿 Medium
Status: ✅ Completed
Tags: CLI
Team: RP, UI, YAT

---

# 🤔 Problem

Many data scientists work in VSCode as their main editor. They use the SSH extension to connect to their VM with one click, bringing their compute resources to their local VSCode terminal. And they are very happy with this workflow! So much so that many people aren’t willing to try Renku unless they can maintain this workflow. In order to pave the way for RenkuLab adoption, we need to support a workflow for using RenkuLab sessions via local VSCode.

Some sample user research from interviews:

* I love VSCode

* Connecting to a RenkuLab Session from VSCode would unify my workflow!

* Using VSCode with a VM is almost like seamless! So mostly I work like that

# 🍴 Appetite

The appetite for this pitch is **2 sprints**, and I am **intentional** about that timeline! This pitch is context dependent on our current place in space-time. 

I believe that enabling a VSCode-based workflow could be a quick win to earn trust with the data scientists we work with, as well as break down a big barrier for adoption. Therefore, I think it’s worth investing serious consideration into how we can break this feature down into steps, and deliver something of value to users *within* the build, not only at its conclusion. 

Therefore, we commit 2 sprints to releasing a polishing solution, with the goal of having intermediate milestones along the way. For example, releasing a version of the solution with no UX to a few beta testers within the first few weeks of the sprint.

# 🎯 Solution

### **CLI-based Session Launch**

From the user point of view, starting a ssh session on RenkuLab should involve only 2 steps:

1. Launch a session on RenkuLab resources from their local terminal via the `renku session start` command
2. Click on the ssh connect button in VSCode that is provided by the VSCode ssh extension to open the session in VSCode.

- An *example* workflow is written out inside this drop down. However, the specific implementation is up to the Build Team! The final design does not need to match this workflow exactly, it’s just a starting point for thinking through how this could work.
    
    **One-Time Initial Set Up** - the user does this one time per project
    
    1. `renku login renkulab.io`
    2. `renku clone <>` (after login, for private project authentication)
    3. `cd <project dir>`
    4.  `renku ssh setup` 
        
        This executes:
        
        - generate a key pair (or offer to use an existing key pair?)
        - add public key to the project repo
        - git push
            - *Note: If protected master, display instructions to add key manually…*
        - Add session entry to ssh config for that key pair (See *Session URLs* in Rabbit Holes below)
        
        (Nice to have: this step launches automatically the first time you run `renku session start` with the `--ssh` flag, so the user doesn’t have to remember to do it separately)
        
    
    **To start a Session:**
    
    1. In a VSCode terminal, `cwd` in the Renku project repo directory where you want to start a session
    2. `renku session start -p renkulab.io <resources> --ssh`. This command lets you know when the session is ready, so you know when to move on to the next step…
    3. Click ssh connect button in VSCode (provided by the VSCode ssh extension)
        1. Or if you’re not in VSCode, and you want to ssh into the terminal, `renku session open **--ssh**`

### Making the Project Session SSH-Compatible

We add ssh to the project base image.

- (Needs SSH server installed and configured, looks for public key in repo and transfers to appropriate location, or knows where to look, etc…)
- Question for Build Team: Is the SSH server *started* on every session, or only ones where it’s requested?

### Communicating which projects are SSH-enabled to users

Enabling SSH connection into a session requires a change to the project docker image. This means that existing projects won’t be SSH-enabled automatically. 

*What does the user have to **do** to make their project SSH enabled?*

- Project → Overview → Status → Update Template
- Note: When we release a new version of the base image, and update the template file, users will get a notification on this status page to update to the newer base image.

*How does the user **know** whether the project is SSH-enabled?*

We have some options here, and I’ve listed a few I’ve come up with so far in the drop down. It is up to the build team to decide which of these options - or other alternatives! - to implement.

- Project SSH Status Indicator Options
    1. Show “Start a session from SSH” option in the Session Start drop down menu, next to “Start with options”. If the project is SSH enabled, the button links you to docs on how to start an SSH session. If project is not SSH enabled, it links you to docs on how to enable ssh for your project.
        
        → Good: it would give a good entry point linking to resources for both:
        
        → Good: No clutter on the Project card/list view
        
        → Bad: could be hard to find
        
    2. All projects have a visual “yes/no ssh” indicator somewhere
        
        → Good: it would give a good entry point linking to resources for both:
        
        “No SSH” → link to docs for how to turn SSH on
        
        “Yes SSH” → link to docs for how to use it
        
        → Bad: but that adds clutter to the UI, and for most projects users don’t care
        
    3. Only show “Yes SSH” indicator, but not a “No SSH” one
        
        → Good: less visual clutter
        
        → Bad: … though eventually most projects will be SSH-enabled anyway, as new projects are created, so eventually this ends up with hardly any less clutter than if there were an icon everywhere.
        

******************************How does the **front-end know** whether the project is SSH enabled?*

The front end can tell if the project is SSH-enabled by checking the template version. 

Note: If we want to make this info available on a project listing (not just on the project page), then may require KG.

Nice to Have: add `ssh-enabled` flag in the template manifest.

# 🐰 Rabbit Holes

### SSH Keys

A simple way to achieve this feature is to create a key pair for the user for the project, and upload the public key to the project.

However, if we rely on keys checked in to the project, this means that anyone who has their key added could potentially ssh in to any running session for that project. This is not a blocking issue for the scope of this pitch, but it’s not *desirable*.

A nice-to-have feature is to create a new key pair every time you launch a session (or other more secure solution).

Questions for the Build Team:

- Q: How much more work is this? Can we accomplish this in 1 sprint?
- Q: Are session-specific keys something we need eventually? If we don’t do this now, how hard will it be to switch to this in the future?

### Whether `renku session start` requires a local clone

In terms of UX, it is acceptable for the scope of this pitch that the user must have a local clone of the repository, to run `renku session start` from.

A nice-to-have feature is to remove this dependency on a local clone, and address the project by the project slug, a la `renku session start rokroskar/my-project`.

### Session URLs

We will have 1 session with ssh support on the project per user, so the url stays the same between sessions, something like this:

`username-group-project.sessions.limited.renku.ch`

Amalthea will simply create another ingress that is well formed like this in addition to the usual name. On a server launch if this ssh-friendly ingress exists the session launch will be aborted.

(FYI This isn’t even a tradeoff, because [Make autosaves not be so bad](https://www.notion.so/Make-autosaves-not-be-so-bad-120264b524ce45d485d407c14a597fb6) is going to restrict sessions to one per project/user anyway)

# 🙅‍♀️ No-gos

The following are out of scope:

### No Go: **Releasing this feature to *all* Renku deployments**

For the scope of this pitch, we only care about having this functionality live on Limited (and nice to have on RenkuLab). All other deployments are not necessary for the scope of this pitch. 
### No Go: Additional SSH session launch entry points

**Browser-based Launch**

1. User starts a session via the browser in RenkuLab
2. User gets a link/credentials/ssh config of some sort
3. User pastes these credentials into VSCode ssh connect
4. … ?
    

### Screenshots of the figma design work in progress… (including information from the issue #2324)

****[lorenzo-cavazzi](https://github.com/lorenzo-cavazzi) commented [3 days ago](https://github.com/SwissDataScienceCenter/renku-ui/issues/2324#issuecomment-1413469186)**

We agreed on showing a new "SSH" option on the "Start" button that opens a modal.The content includes links to documentation and how to enable the project to support SSH (or how to proceed).

---

![Untitled](Support%20RenkuLab%20compute%20access%20from%20local%20termina%20f896d3b391c94bcc87c56e375eb531d6/Untitled.png)

![Untitled](Support%20RenkuLab%20compute%20access%20from%20local%20termina%20f896d3b391c94bcc87c56e375eb531d6/Untitled%201.png)

## 

SSH Project Status Indicator , label ideas for the ssh support

![Untitled](Support%20RenkuLab%20compute%20access%20from%20local%20termina%20f896d3b391c94bcc87c56e375eb531d6/Untitled%202.png)

here is an example of the design template for the modal. Could someone please send me screenshots or links for the information that should be displayed on it ? ..

![Untitled](Support%20RenkuLab%20compute%20access%20from%20local%20termina%20f896d3b391c94bcc87c56e375eb531d6/Untitled%203.png)

![Screenshot 2023-02-27 at 16.31.22.png](Support%20RenkuLab%20compute%20access%20from%20local%20termina%20f896d3b391c94bcc87c56e375eb531d6/Screenshot_2023-02-27_at_16.31.22.png)