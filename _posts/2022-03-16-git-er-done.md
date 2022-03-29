---
title: "Git 'Er Done!"
subtitle: "Git Workflows That Help You Move Faster"
date: 2022-03-28T14:30:00-04:00
header:
  teaser: /assets/images/posts/git/git.jpg
categories:
- blog
tags:
- git
- source control
- software
- continuous integration
- devops
---

Gitflow, Forking, Feature Branch, and Centralized are all Git workflows. But which one can help you deliver stable working software the fastest? 
In this post, we're going to dive into each of these workflows, pop the hood and discuss what's underneath.

![Continuous Integration](/assets/images/posts/git/git.jpg)

_<small>Image by [gdainti](https://www.istockphoto.com/portfolio/gdainti?mediatype=illustration) on [iStock](https://www.istockphoto.com/) by Getty Images</small>_

A git workflow is an agreed upon way team members interact
with Git. Git's flexibility means that there are several workflows to choose from. Let's unpack some of the more
common Git workflows.

### Contents

* [Centralized Workflow](#centralized-workflow)
* [Feature Branch Workflow](#feature-branch-workflow)
* [Forking Workflow](#forking-workflow)
* [Gitflow Workflow](#gitflow-workflow)
* [What About Mainline Development?](#what-about-mainline-development)

### Centralized Workflow

![Centralized Workflow](/assets/images/posts/git/centralized.png)

#### How It Works

The benefits of continuous integration become super obvious when centralized development is used. This is because
everything is pushed to a single branch. So, conflicts must be resolved on every push. Additionally, every push
must be validated since any push could have the unintentional side effect of introducing build breaking changes to
every other team member.

Here's how it works:

1. Pull the latest updates from the main branch. <small>(```git pull```)</small>
2. Make a change.
3. Commit the change. <small>(```git commit -m "some change msg"```)</small>
4. Push the change. <small>(```git push origin main```)</small>

Easy peasy... except when step #4 reveals a merge conflict. In that case, you would then pull down the latest changes
(i.e. merge them into your local), resolve any merge conflicts, and then push the changes.

#### Benefits

A centralized workflow is one of the simplest workflows, if not the simplest workflow to understand. 
Every commit by every person on the team is based off of, and integrated into a single _main_ branch. The 
obvious benefits of a centralized workflow are

* Everyone works off of the same branch, so drift is naturally minimized
* Merging is a regular practice that happens all the time (as opposed to working in batches where integrations can be extremely painful)
* There is one place to look to see the current development stream, releases, history, etc.

#### Drawbacks

As great as centralized workflows are, there are some drawbacks.

* Cohesive work items may not be well-represented in the commit log.
* Immature or lower performing teams may struggle with centralized development workflows.
* It can be challenging to support external/non-trusted contributors.

#### How Releases Work

Releases for centralized development can be based on tags. For example, tagging a commit with ```v1.0``` would release
the ```v1.0 release```, which could be done through a deployment pipeline. This could either be a fully automated release,
or it could be stepped through a delivery pipeline with human workflow elements (i.e. approvals and the like).

Another way to do releases is to make every commit publish a release through a deployment or delivery pipeline. This approach
requires a mature team and pretty solid continuous integration practices in addition to a well-thought-out deployment or delivery pipeline.

Both of the above approaches can mean that incomplete features are shipped, which may be undesirable: imagine a shopping
cart that is missing checkout functionality. That is probably not a desirable situation. However, there is a solution! And, that
solution is [feature flags](https://martinfowler.com/articles/feature-toggles.html). Without getting too far down the road 
on how to implement feature flags (possibly in a future post?),
a feature flag allows you to ship parts of a feature that remain "switched off" until the remaining parts of the feature
are shipped. At that time, the feature flag is toggled and the entire feature is available all at once. Pretty neat, huh?

### Feature Branch Workflow

![Feature Branch Workflow](/assets/images/posts/git/feature-branch.png)

Unlike trunk-based, which is a "never branch" workflow, feature branch workflows are "always branch."

#### How It Works

1. Pull the latest updates from the main branch. <small>(```git pull```)</small>
2. Checkout a new feature branch from main. <small>(```git checkout -b my-new-feature```)</small>
3. Make a change.
4. Commit the change. <small>(```git commit -m "some change msg"```)</small>
5. Merge down from main. (```git merge origin/main```)
6. Correct any merge conflicts.
7. Push the change. (```git push origin my-new-feature```)
8. Repeat steps 3-7 until the feature is complete.
9. Merge ```my-new-feature``` branch into main.

#### Benefits

Feature branch workflows isolate changes associated with a feature into a single branch. This means that
all changes associated with a feature are autonomous - they are merged or not merged as a single unit. It also
means that work items can be tracked by branch, without the need for special commit messages.

Feature branch workflows also isolate breaking builds to the feature branch. If you have an immature team or a situation 
where team members may frequently break the build, those consequences are felt mostly in the feature
branches and not in the main branch.

* Feature branching makes work items autonomous
* Work can be easily tracked by feature/branch
* Builds break mostly in feature branches

#### Drawbacks

The drawbacks associated with feature branch workflows are as follows:

* Multiple feature branches in flight may cause batch integration problems on merge into main
* Feature branches that run long have a serious risk of drifting apart from each other

However, I would assert that those drawbacks are not insurmountable and that they don't necessarily mean that
feature branching can't work with continuous integration. It does mean that some careful attention must be taken
in order limit the significance of the aforementioned drawbacks. For example, features should be small 
enough that they can be completed and integrated into the main branch within a single day.

#### How Releases Work

Similar to the centralized workflow above, releases can either be triggered off of tags or on every merge into the mainline branch
from a feature branch. Some teams choose to make special branches for a variety of reasons. 
Resist this urge! Different branches diverging from the mainline are a recipe for snowflakes. 

Release branches (aka release lines) are OK, so long as their purpose is clear, and they are merged back into the main
branch in short order.

### Forking Workflow

![Forking Workflow](/assets/images/posts/git/forking-workflow.png)

Forking workflows, like feature branching, mean that work is done in separate branches. However, unlike feature branching,
those branches are in different forked repositories. Forking workflows are a good way to integrate code from untrusted parties.

#### How It Works

1. Fork a new repository from a server-side repository.
2. Make a change.
3. Commit the change. <small>(```git commit -m "some change msg"```)</small>
4. Merge down from the upstream repo. (```git pull {upstream repo} {branch name}```)
5. Correct any merge conflicts.
6. Push the change. (```git push origin branch```)
7. Repeat steps 3-6 until the feature is complete.
8. Create a pull request to merge the branch on the forked repo into a branch on the upstream repo.
9. The pull request is evaluated by the team owning the upstream repository.
10. If accepted, the code in the forked branch is merged into the upstream repository's branch.

#### Benefits

As stated above, forking allows submissions from untrusted parties, which is a pretty common scenario in open source
projects. Additionally, within an organization, accepting submissions from people on other teams can also be achieved through a 
forking workflow. Finally, a forking workflow can be the way a team does work, or it can be supported in addition
to some other workflow, like a feature branch or a centralized workflow.

#### Drawbacks

* Some level of effort is required to review pull requests from forked branches as a batch
* Forking workflows may feel slightly more complicated than other types of workflows

#### How Releases Work

Think of forking as being like feature branching, but across different repositories. Additionally,
because forking can be used for the very specific purpose of accepting untrusted code submissions,
it's easy to see that a forking workflow can be combined with another workflow. This means that the
release approach can easily be adopted from another workflow, like a centralized workflow or a feature branch
workflow.

### Gitflow Workflow

Once, universally accepted as the most popular Git workflow, Gitflow has recently fallen out
of favor due to its relative complexity compared to other Git workflows, its promotion of long-running branches,
and the out-of-band changes it advocates that can make for challenging merges. However, Gitflow and derivatives of it
are still used.

#### How It Works

A Gitflow workflow has two long-running branches: main and develop. The main branch holds the release history, with
the HEAD of the main branch always pointing to the code deployed to the current production release. The develop branch
represents the current development _stream_. The HEAD of the develop branch always points to the most recently completed
code under development. 

Gitflow has essentially three workflows: a feature development workflow, a hotfix workflow and a feature release workflow. They are as follows.

##### Gitflow Feature Branches

![Gitflow Features](/assets/images/posts/git/gitflow-feature.png)

1. Checkout a new feature branch from develop. <small>(```git checkout -b feature/{feature name}```)</small>
2. Make a change.
3. Commit the change. <small>(```git commit -m "some change msg"```)</small>
4. Merge down from develop. (```git merge origin/develop```)
5. Correct any merge conflicts.
6. Push the change. (```git push origin feature/{feature name}```)
7. Repeat steps 3-6 until the feature is complete.
8. Merge ```feature/{feature name}``` branch into develop.

##### Gitflow Hotfixes

![Gitflow Hotfixes](/assets/images/posts/git/gitflow-hotfix.png)

1. Checkout a new hotfix branch from main. <small>(```git checkout -b hotfix/{hotfix name}```)</small>
2. Make a change.
3. Commit the change. <small>(```git commit -m "some change msg"```)</small>
4. Merge down from main. (```git merge origin/main```)
5. Correct any merge conflicts.
6. Push the change. (```git push origin hotfix/{hotfix name}```)
7. Repeat steps 3-6 until the hotfix is complete.
8. Merge ```hotfix/{hotfix name}``` branch into main & develop.
9. Tag main as a patch release.

##### Gitflow Feature Releases

1. Merge the develop branch into the main branch.
2. Tag the main branch as either a minor release or a major release.

#### Benefits

* Release-oriented workflow
* The main branch stores an easy-to-follow ordered release history in its commit log.
* The develop branch stores an easy-to-follow ordered history of work items.
* Gitflow is naturally compatible with semantic versioning.
* The workflow can have its events easily integrated into common development practices like...
   * publishing release candidates
   * publishing final releases
   * publishing hotfixes independent of in-progress development work

#### Drawbacks

* Potentially complex merge integrations
* More than one long-running branch complicates the workflow

Sure, there are only a few drawbacks listed. But, they are big ones. They are arguably so big that they negate
any value that might be achieved by maintaining more than one long-running branch (develop & main).

##### Consider the following:

A Gitflow feature release means that the develop branch is merged into the main branch. That seems easy enough, except that
you want every commit in the main branch to represent a release; that makes things nice & tidy. But, that also means
that the main branch and the develop branch do not share a commit history. And, because they do not share a commit history, the
merge is going to be really painful. 

So, now you decide that the main and the develop branches will share a commit history (i.e. you won't squash merges into
main or anything of the like). Now, when you want to merge the develop branch into the main branch, it's easy. Git recognizes
the matching commit history and applies the deltas. Yay, problem solved! But is it?

Now that develop and main share the same commit history, what additional value does the main branch provide over just
using a single long-running branch (i.e. a trunk)? After all, you don't need two branches to tag releases. The only advantage 
having a main branch and a develop branch has now is that it makes hotfix branching more straightforward, but only slightly.

By having a main branch and a develop branch, it allows hotfix branches to be created off of the head of the main branch. However,
if a single trunk represented both the main and develop branches, then it wouldn't be that hard to create a hotfix branch off of
the last release tag. Or better yet, add hotfixes to the head and make use of 
[feature flags](https://martinfowler.com/articles/feature-toggles.html) to ensure that any functionality not ready to 
be shipped is turned off until it is ready. 

Oh, wait! We just turned Gitflow into a feature branch workflow. :)

#### What About Mainline Development?

> [Developing on mainline] is an extremely effective way of developing, and the only one which enables you to perform
> continuous integration.
>
> <small>_-Continuous Delivery (Humble, Farley)_</small>

The above statement is sometimes used as an argument for a [centralized workflow](#centralized-workflow). However,
mainline development (aka trunk-based development) does not necessarily equate to a centralized workflow. Mainline 
development can be achieved using a centralized workflow. But, it can also be achieved using other workflows, too.

> Doing mainline development does not mean "do not branch." It means that all on-going development activities end up on
> a single codeline at some time
>
> <small>_-Software Configuration Management Patterns (Berczuk, Appleton)_</small>

Given that, any of the above Git workflows can be mainline development workflows if...

1. Only a single branch (i.e. the main branch) is long-lived
2. Any other branches are (a) short-lived and (b) clear in their intent
3. Short-lived branches are merged back into the main branch in a timely manner (< 1day)

As far as which workflow to use, that's up to the team!

---

### Resources

* [Continuous Integration by ThoughtWorks](https://www.thoughtworks.com/continuous-integration)
* [Feature Toggles (aka Feature Flags)](https://martinfowler.com/articles/feature-toggles.html)
* [Distributed Git - Distributed Workflows](https://git-scm.com/book/en/v2/Distributed-Git-Distributed-Workflows)
* [Feature Branch](https://martinfowler.com/bliki/FeatureBranch.html)
* [A Successful Git Branching Model](https://nvie.com/posts/a-successful-git-branching-model/)
* [Continuous Delivery](https://www.amazon.com/Continuous-Delivery-Deployment-Automation-Addison-Wesley/dp/0321601912)
* [Software Configuration Management Patterns](https://www.amazon.com/Software-Configuration-Management-Patterns-Integration/dp/0201741172)
