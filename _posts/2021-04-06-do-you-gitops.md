---
title: "Do You Even GitOps?"
date: 2021-04-06T15:30:30-04:00
header:
  teaser: /assets/images/posts/gitops/gitops.jpg
categories:
  - blog
tags:
  - git
  - gitops
  - devops
  - automation
---

GitOps is all the rage. You are either using it or you want to be using it. But what is it? And why does it matter? This post
breaks down what GitOps is and why it matters.

![GitOps](/assets/images/posts/gitops/gitops.jpg)

_<small>Image by [gdainti](https://www.istockphoto.com/portfolio/gdainti?mediatype=illustration) on [iStock](https://www.istockphoto.com) by Getty Images</small>_

### What is GitOps? ###

In the ACM article _[GitOps: A Path to More Self-service IT](https://queue.acm.org/detail.cfm?id=3237207),_ it is asserted that IaC + PR = GitOps. But we think it's 
more than that. So, we offer up the following alternative definition.

> GitOps is build, test and release automation attached to logical events in a Git workflow.

### Git Workflows ###

Commonly used Git workflows include Trunk-Based Development, Git Flow and others. For the sake of brevity,
we will only touch on Trunk-Based Development and Git Flow workflows.

#### Trunk-Based Development ####

With a Trunk-Based Development workflow, there is a single _main_ branch that is used to create short-lived branches
for development (usually lasting no longer than a day). Following a small batch of commits, a Pull Request is raised
and following a code reivew, the short-lived branch is merged into the _main_ branch.

Alternatively, some developers engage in pair programming (as a kind of continuous code review) and commit directly to the _main_ branch.

When it comes time for a release, a release branch is created to shield the pending release from incoming 
commits that are not included in the release and to create a record of the release. Any bugs fixed related to the release
are made in the _main_ branch and cherry picked into the release branch. Once the release is complete, the release
is tagged.

An alternative to having a release branch that is sometimes employed is releasing and tagging directly from the _main_ branch.

#### Git Flow ####

Git Flow has 2 long-lived branches: develop and master (or main). Feature branches are created off of the develop branch,
and they are merged back into the develop branch via a Pull Request once the feature is complete.

When a release is imminent, a corresponding release branch is created and any changes made to the release branch
prior to the release are also merged into the develop branch. Once the release is shipped, the release branch
is merged into both the master and develop branches. The release is then tagged in the master branch.

The develop branch, therefore, represents the current development stream (completed work that has not yet been released).
And the master branch has the current release at its head, while each of its other tags correspond to a past release.

More details on Git Flow can be found in Vincent Driessen's post, [A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/).

### GitOps with Git Flow ###

Git Flow is the obvious choice to illustrate GitOps through examples since GitOps is somewhat ceremonious (lots of events).
That shouldn't necessarily be viewed as an endorsement of GitOps. Trunk-Based Development has a number
of merits. However, the scope of this post is GitOps. So, a discussion of those merits will have to wait
for a future post.

Let's take a look at some of the events associated with Git Flow

1. PR raised to merge into develop branch
2. Merge into develop branch
3. Release branch created
4. Release branch updated
5. Merge into master branch

Now, let's associate each of those events with build & deployment actions

1. PR raised to merge into develop branch (build/test)
2. Merge into develop branch (build/test/dev release)
3. Release branch created (RC release)
4. Release branch updated (build/test/RC release)
5. Merge into master branch (release)

By associating Git workflow events with build & deployment actions, self-service through the agreed upon
Git workflow is achieved. That's GitOps!

These workflow event action associations can work whether the project is a runnable application that has its infrastructure
defined using IaC or whether it's a published artifact (e.g. Maven or PyPI).

### A GitOps Example Using Git Flow ###

The development of a new feature begins when a developer creates a feature branch from the long-lived develop branch.
Once the feature is complete, a developer raises a Pull Request to merge the feature branch into the develop branch.
The creation of the Pull Request triggers a merge check (is it able to be merged?) and it kicks off a build.
Assuming that the feature branch is able to be merged into the develop and the build passes, a code review is conducted.
Upon completion of a successful code review, the feature branch is merged into the develop branch, which then 
triggers another build and a deployment to a shared development environment.

After some time, a release is needed. So, a release branch is created from the develop branch, which triggers the
deployment of a release candidate to a staging environment. Here, the release is vetted prior to actually being released.
If there is a bug that needs to be fixed in the release, then that would trigger a build & release candidate deployment when 
the release branch is updated. It would also trigger a build & deploy to a shared development environment when those updates
are merged into the develop branch.

Once the release is ready, the release branch is merged into the master branch. On that merge event, the release candidate
is deployed as a release.

### Implementing GitOps ###

The automation tied to Git events that allows GitOps to work can be implemented with Continuous Integration (CI)
tooling, which is designed to trigger actions based on Source Control Management (SCM) events. Tools like [Jenkins](https://www.jenkins.io/),
[TeamCity](https://www.jetbrains.com/lp/teamcity/cloud-enterprise/?gclid=Cj0KCQjwsLWDBhCmARIsAPSL3_2HmOhxgTrrOo6GLYdhantZ_xtKqY8diBeqLWy_1i-7ytHBZJOIdfUaAgOTEALw_wcB&gclsrc=aw.ds),
or [GitHub Actions](https://github.com/features/actions) are reasonable candidates.

To ensure that all artifacts are available and working with the current versions of the application source
stored in Git, any configuration items must also be checked into SCM, along with IaC for supporting infrastructure.

### Summary ###

* GitOps is build, test and release automation attached to logical events in a Git workflow.
* GitOps provides self-service by tying deployment actions to Git workflow events.
* All artifacts required to build, test and deploy (i.e. configuration & IaC) are checked into SCM (Git).
* Continuous Integration (CI) tooling can be used to associate actions (e.g. build, test, deploy) with Git events.
