---
title: "Continuous Integration (CI) Explained"
subtitle: "What it is, what it does, and how to implement it"
date: 2022-02-08T15:00:00-04:00
header:
  teaser: /assets/images/posts/ci/iStock-1137632555.jpg
categories:
- blog
tags:
- codezero
- software
- continuous integration
- devops
---

Software development automation != Continuous Integration (CI). However, the former is a component of the latter. 
Here, we focus specifically on CI: what it is, what it does, why it's important and how to implement it.

![Continuous Integration](/assets/images/posts/ci/iStock-1137632555.jpg)

_<small>Image by [Natali_Mis](https://www.istockphoto.com/portfolio/Natali_Mis?mediatype=photography) on [iStock](https://www.istockphoto.com/) by Getty Images</small>_

> Continuous Integration (CI) is a development practice that requires developers to integrate code into a shared repository several times a day. 
> Each check-in is then verified by an automated build, allowing teams to detect problems early.

_<small>-[Thoughtworks](https://www.thoughtworks.com/continuous-integration)</small>_

### Contents

* [What Is CI?](#what-is-ci)
  * [Feedback Loops](#feedback-loops)
  * [Feedback From Automated Builds](#feedback-from-automated-builds)
* [Why Is CI Important?](#why-is-ci-important)
* [Implementing CI](#implementing-ci)
  * [Frequent Integration](#frequent-integration)
  * [Automate The Build](#automate-the-build)
    * [Example: CodeZero](#example-codezero-by-blu-flame-technologies) 
* [TL;DR](#tldr)

### What Is CI?

As stated by the Thoughtworks' quote above, CI is a development practice that requires developers to integrate their code
at least once, but ideally multiple times per day. Each integration is then verified by an automated build, which
provides a short feedback loop.

As direct as that description of CI is, there is a lot to unpack, like...

* What is a feedback loop?
* What exactly should be included in an automated build?
* How can you practically integrate code multiple times per day?

#### Feedback Loops

A feedback loop is anything that gives you useful information. That information can then be used, in turn to
determine whether if it's appropriate to stay the course or if a course corrections is needed.
The longer a feedback loop is, the longer you may be going down a bad path without the necessary information to know how, 
or even if you need to make a correction.

Examples of feedback loops:

* code compilation
* code linting
* static code analysis
* automated tests
* input from a Product Owner
* code deployment
* higher level manual tests
* input from customers

As you probably noticed, some of the above examples could be combined into a single feedback loop. For example, code
linting and static code analysis could be included in the same feedback loop that gives you multiple data points.

You probably also noticed that some feedback takes longer than other feedback. For instance, a developer likely gets
regular feedback on code compilation. However, feedback from manual testing certainly takes longer, and is therefore
less frequent. We would then say that code compilation has a short feedback loop, while manual testing has a longer
feedback loop.

#### Feedback From Automated Builds

Like many things in software development, determining what feedback should be provided on automated builds triggered
on code integration is both an art and a science. OK, it's not really much of either. But there are some rules/guidelines/heuristics
that can be applied to let us know whether the feedback from an automated build is at the appropriate level.

**1. Build Time**

Automated builds are supposed to have relatively short feedback loops. They are being executed multiple times per day.
So, it wouldn't make much sense to wait an hour or more to see whether an automated build successfully completed or not.
Shorter is generally better. However, short with better information is better than shorter with very little information.

As a guide, a build time of [10min or less](https://www.jamesshore.com/v2/books/aoad1/ten_minute_build) is generally considered pretty good.

**2. Linting & Static Code Analysis**

Both code linting and static code analysis feedback can provide information about code quality, with code readability
being a component of code quality. Sure, they don't give as much insight into the working state of the software as 
more robust tests. But, they do provide useful information, and they do it fast. That's why code linting and static 
code analysis are great candidates to include in automated builds.

**3. Tests**

> Include as much testing as is feasible while still keeping your automated builds fast.

There are [many different types of tests](https://blog.bluflametech.com/blog/test-software/). Some tests, like unit tests 
are pretty granular, but they can be executed quickly. Other tests, like broad scoped acceptance tests may provide a better
picture of the complete system, but they can require prerequisite scaffolding; they can also be slow to execute. 

It stands to reason that since unit test suites can execute a large volume of tests in a relatively short period of time,
they should be a natural candidate for inclusion into automated builds. Indeed, that is good thinking! However, increasingly,
the line between unit tests and more robust integration tests has been blurred. This is evidenced by testing frameworks
like those produced by [Spring](https://spring.io/guides/gs/testing-web/). Unit tests offer a higher volume of tests
that can be executed quickly and integration tests offer a lower volume of tests executed at a slower rate, but with
potentially more information about the working state of the system.

Odds are that tests beyond the scope of integration tests will lengthen the feedback cycle of the automated build to
the point of being significantly less useful. But, as far as which tests should be included in your automated builds...
Include as much testing as is feasible while still keeping your automated builds fast. You want useful information/validation,
but you also want that information quickly. Stale information is not useful information.

**4. Deployment**

Depending upon the type of software being developed, the deployment of that software may be included in the automated
build. For example, if the software being developed is Infrastructure as Code (IaC), then you really have no idea if it works or what it does
until after it is deployed. However, if you are writing a robust Python application then a complete deployment 
may be more time-consuming than it is useful, especially when compared to the other information supplied by the build 
(i.e. compilation, linting, static code analysis, unit tests).

CodeZero is a hosted software development automation by Blu Flame Technologies. It provides Continuous Integration and
Continuous Delivery services. Additional metrics, product integrations and self-service capabilities will follow. But
for right now, its base is validate/build/test/deploy with low maintenance overhead and near instant project on-boarding. 

### Why Is CI Important?

> By integrating regularly, you can detect errors quickly, and locate them more easily.

_<small>-[Thoughtworks](https://www.thoughtworks.com/continuous-integration)</small>_

CI provides short, frequent feedback loops. And, because CI executes on every integration, it's easy to discern
when a problem was introduced and what code was the culprit. Sure, CI is not going to catch everything. But it can
catch a lot; it's a great practice that can go a long way towards improving the stability of the codebase.

An important aspect of CI is the automated build that executes in an environment detached from any human's working
environment. This eliminates _works on my machine_ build validation and ensures a consistent, repeatable build that
can be universally understood.

### Implementing CI

> The component of CI that teams seem to struggle with the most is frequent integration.

#### Frequent Integration

The component of CI that teams seem to struggle with the most is frequent integration. On some teams, developers  
like to hide their code until they are "ready" before submitting a pull request and blasting any would-be reviewers
with mountains of code changes. This practice is related to [Hero Culture](https://hbr.org/2021/03/does-your-company-lurch-from-crisis-to-crisis),
and it creates a ton of problems for the team.

So, the first practice that **the entire team** must adhere to is **every developer** integrates **every day**.

This practice may bring to light some other team practices that inhibit frequent integration, like the code review process. For example,
if your goal is to integrate every day, but in order to integrate you must have a group code review, which are scheduled
every other day then daily integration is not going to happen because the code review process won't let it happen.

Some alternatives to scheduled group code reviews:

* Peer reviews - one other team member must review prior to integration
* [Pair](http://www.extremeprogramming.org/rules/pair.html) or [mob programming](https://www.agilealliance.org/glossary/mob-programming) - 2+ team members code it and review it
* Daily code reviews - daily scheduled group or small audience code reviews

Daily code reviews ensure that at best, code will be integrated once per day. So, the minimum requirement has been met,
but at what cost to the team? Pair programming and peer reviews can see code integrated with more frequency, with pair or mob programming
having the edge since at least two developers are invested as opposed to one developer chasing down others for a code review.

It should also be noted that peer reviews and pair or mob programming do not necessarily mean that group code reviews cannot happen.
If the team feels that group code reviews are beneficial then they can still occur outside the critical path. The point is that code reviews should
not impede integration frequency.

#### Automate The Build

Automating the build should be amongst the very first steps of any new software project. It should also be noted that 
the build is runnable on both a developer's workstation and on the CI/Build server. However, the source of truth for the automated build is
the CI/Build server.

##### Example: CodeZero By Blu Flame Technologies

Setting up an automated build for GitHub projects with [CodeZero](https://codezero.bluflametech.com) could not be easier!

After [signing in to CodeZero](https://codezero.bluflametech.com/docs#signing-up), click the **+** box on the 
Projects page to [add a GitHub repo as a CodeZero project](https://codezero.bluflametech.com/docs#adding-a-project).

Once the project's card is visible, CodeZero is ready to listen for events. Now, a [**.codezero.yml** configuration](https://codezero.bluflametech.com/docs#configuration)
is needed in the root of the corresponding repository to let CodeZero know which events to listen for and what to do
when it gets them.

Let's walk through the **.codezero.yml** configuration of our forked terraform-maven repository.

First, the [language and version](https://codezero.bluflametech.com/docs#supported-languages) are specified. Since terraform-maven is a Java 8 project, the language configuration looks like this.

```yaml
lang:
  java: 8
```

Next, the events that trigger the build automation are specified. The current event config in terraform-maven's **.codezero.yml** file 
tells CodeZero to execute a build on a push commit/merge to any branch. It looks like this.

```yaml
events:
  git_branch: any
```

If we wanted to change that to only execute a build on a push commit/merge to the _main_ branch, the updated configuration would look like this.

```yaml
events:
  git_branch: main
```

Next, we can see that ops (short for operations) are defined. Operations are cohesive operations that can be grouped together into stages.
Our terraform-maven **.codezero.yml** has two operations defined (_setup.terraform_ and _test.tf-build-tools_). Operations are named as {logical category}.{operation/module}.

```yaml
ops:
  setup:
    - terraform:
        path: ''
        script: |
          sudo apt-get -qq install -y unzip
          curl -g -s https://releases.hashicorp.com/terraform/0.12.24/terraform_0.12.24_linux_amd64.zip -o /tmp/terraform.zip
          sudo unzip -q /tmp/terraform.zip -d /usr/bin
  test:
    - tf-build-tools:
        script: ./mvnw clean test -no-transfer-progress
```

Under setup.terraform, there exist two elements: path and script. By default, the path of an operation is the name of its 
module under the root of the repository. Since, the setup.terraform operation is to be run in the root of the repository
and not under a ./terraform directory, the default path is overridden. The script element defines how to source and
install Terraform 0.12.24 for the build.

Under the test.tf-build-tools operation, there is no path element because its script is expected to be executed in the ./tf-build-tools
directory.

Finally, the stages defined in the repository's **.codezero.yml** organize the execution of the operations. Stages are
executed in order.

```yaml
stages:
  - setup:
      ops:
        - setup.terraform
  - build:
      ops:
        - test
```

Looking at the stages configuration above, there are two stages defined: setup and build. The setup stage executes the
setup.terraform operation; it uses the fully qualified operation name. 

The build stage executes the test operation, but it does not supply the complete operation name. This tells CodeZero
to execute every operation in order under 'test.' Since there is only one operation (tf-build-tools), the build stage will execute
the test.tf-build-tools operation.

###### Putting It All Together

```yaml
lang:
  java: 8
events: 
  git_branch: main
ops: 
  setup:
    - terraform:
        path: ''
        script: |
          sudo apt-get -qq install -y unzip
          curl -g -s https://releases.hashicorp.com/terraform/0.12.24/terraform_0.12.24_linux_amd64.zip -o /tmp/terraform.zip
          sudo unzip -q /tmp/terraform.zip -d /usr/bin
  test:
    - tf-build-tools:
        script: ./mvnw clean test -no-transfer-progress
stages:
  - setup:
      ops:
        - setup.terraform
  - build:
      ops:
        - test
```

The configuration above, when triggered by a change to the _main_ branch, bootstraps the build with Java 8, downloads and installs terraform 0.12.24 and then executes the
maven build inside tf-build-tools.

### TL;DR

* Continuous Integration (CI) is a software development practice that... 
  1. requires developers to integrate their code into a common branch at least once per day, but ideally multiple times per day
  2. verifies each integration by an automated build in a short feedback loop
* A feedback loop is anything that gives you useful information which can then be used to determine if it is appropriate to stay the course or if a course correction is needed.
* Validation/analysis, builds, tests and deployments are all eligible for inclusion in automated builds.
  * Keeping the build fast is important, however. So, choose the fastest & highest value operations.
* Shorter integrations aside, CI also has the benefit of highlighting errors shortly after they occur, allowing them to be targeted and fixed quickly.
* Frequent integration is the cornerstone of CI; it's also something that teams seems to struggle with.
  * CI tooling != CI
  * Everyone on the team must participate; CI is a culture.
  * Code reviews may be hampering adoption of CI; adjust, so they don't impede integration frequency.
* Automating the build should be amongst the very first steps of any new software project.
  * [Example: CI with CodeZero by Blu Flame Technologies](#example-codezero-by-blu-flame-technologies)

---

### Resources

* [Continuous Integration by ThoughtWorks](https://www.thoughtworks.com/continuous-integration)
* [Hero Culture by Harvard Business Review](https://hbr.org/2021/03/does-your-company-lurch-from-crisis-to-crisis)
* [Pair Programming by Extreme Programming](http://www.extremeprogramming.org/rules/pair.html)
* [Mob Programming by Agile Alliance](https://www.agilealliance.org/glossary/mob-programming)
* [CodeZero by Blu Flame Technologies](https://codezero.bluflametech.com)