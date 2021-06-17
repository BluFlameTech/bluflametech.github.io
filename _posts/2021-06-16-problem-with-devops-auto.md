---
title: "The Problem(s) With DevOps Automation"
date: 2021-06-16T09:34:30-04:00
header:
  teaser: /assets/images/posts/problem_with_devops_auto/frustrated_programmer.jpg
categories:
- blog
tags:
- automation
- devops
---

Automation is a significant component of what allows DevOps teams to work at breakneck speeds.
Unfortunately, that nirvana isn't often achieved by many teams. Why is that?!?!

![Frustrated Programmer Using Modern DevOps Tooling](/assets/images/posts/problem_with_devops_auto/frustrated_programmer.jpg)

_<small>Credit: Getty Images</small>_

> Success in software development requires that organizations empower developers, enabling them to build productively, collaborate globally & securely, and scale innovation.

_<small>Microsoft & Sogeti Enterprise DevOps Report 2020-2021</small>_

First, there was build automation, then there was continuous integration (CI), then there was continuous
delivery (CD) and infrastructure as code (IaC) and GitOps...

Now, there are a ton of automation tools, all with the promise of accelerating DevOps practices. So,
why do we still have a _Sprint Zero_ with automation onboarding aspirations that are rarely achieved? Why do we spend as much
time or more developing automation software than the software it's supposed to automate? How much of that actually
empowers and enables developers to be more productive? Probably not as much as we like to claim.

### What is DevOps?

Before we go any further, we should probably get on the same page in terms of our definition of DevOps. Below are some
definitions of DevOps.

> The union of people, process and technology to enable continuous delivery of value to customers.

_<small>Microsoft</small>_

>The architecture, technical practices, and cultural norms that enable us to…
> increase our ability to deliver applications and services...
> quickly and safely, which enables rapid experimentation and
> innovation, and the fastest delivery of value to our customers…
> while ensuring world-class security, reliability, and stability...
> …so that we can win in the marketplace.

_<small>Gene Kim, DevOps Days</small>_

> DevOps represents a change in IT culture, focusing on rapid IT service delivery through the adoption of agile, lean practices in the context of a system-oriented approach.

_<small>Gartner</small>_

Given the above definitions, we now offer our own definition of DevOps.

> A multi-disciplinary approach to quickly and iteratively deliver value-adding software systems; DevOps is predicated on the unification of software development (Dev) and IT operations (Ops).

_<small>Blu Flame Technologies</small>_

### DevOps Automation

A big part of the _quick_ and _rapid_ aspects of DevOps is automation. 20 years ago, the "it" software development automation tool was CruiseControl, then it 
was Hudson and Jenkins. Now, there's TeamCity, TravisCI, GitHub Actions, CircleCI and more. The interfaces have gotten 
better, and the functionality has improved. But, at their core they are continuous integration (CI) tools. They mostly
trigger off of source control events; what they do and how they do it is programmed by the developer. 

---

#### Problem 1: Onboarding

Some CI tools make onboarding new projects easier than others. GitHub Actions and TravisCI, for example
have relatively short project onboarding overhead. Simple streamlined projects could likely get up and running within a day or less,
with TravisCI trending more towards faster onboarding due to their convention over configuration approach. However,
more complex project configurations quickly see that low project onboarding overhead evaporate.

At the other extreme, Jenkins favors heavily bespoke project builds through its robust extension options. Jenkins allows for pipelines to 
be scripted in their own Groovy-based _CPS_ language, which can hook into Java libraries and even source them from a repo
under certain conditions using Grape. Jenkins also allows the use of custom plugins, which is not uncommon in the space.
However attractive the prospect of writing oodles of custom Jenkins code may be, the drawbacks are even more significant.
Organizations who go down that route no doubt know the pain inflicted on project timelines by spending a significant amount
of development cycle time updating and fixing build and deployment automation at the expense of building out their software
systems.

The choice is then between extreme flexibility and fast project onboarding, with various product offerings vying for space in the middle.
But in general, the development team is either incentivized to roll their own automation with significant and granular 
customization, or they are encouraged to abide by conventions for the sake of simplicity. That's not to say tools that favor simplicity, like 
TravisCI can't be extended. But extension is definitely an afterthought as the penalty for more robust build automation 
is a steep complexity curve.

---

#### Problem 2: Scaling Complexity

Not suprisingly, many projects start off as simple and then become more complex as time goes on. They become more robust,
accumulate more technical debt, and their builds & deployments mirror the increase in project complexity. Another interesting
thing is that hosted builds executed by a build server (aka CI server or build automation server) become more disconnected
from the builds developers execute on their local workstations. The size of that gap depends on the complexity of the builds.
The more painful and time-consuming the builds become, the less frequently they are executed, and the bigger the disconnect
becomes.

However, even when build complexity is managed well, there is nearly always a gap since what the developer executes on his
or her workstation for local build/test/deploy is not exactly what is executed on the build server. Even when using Kubernetes
or Docker Compose or a local build server, the gap remains - not exactly the same is still not exactly the same. 

Just as water always flows to the lowest point, people do what they are incentivized to do. If running a local 
build tool (e.g. Gradle, Make) instead of the official project build is the path of least resistance for quick iteration, that's what will happen. 
But eventually, the build must successfully execute on the build server. If it's slow, complex or painful 
to execute then that pain will get pushed to the last possible moment, where it will be the most costly to correct. At 
that point, the build automation that was intended to shorten development cycle times becomes something that lengthens them.

---

#### Problem 3: Build Orchestration

Whether you have a [Monorepo](/blog/monorepo/) or multiple independently deployable project repositories, orchestrating
complex builds and deployments isn't easy. Sadly, for most build automation servers, it's an afterthought. Because of this, 
and especially for builds, it's become increasingly common to use a local project-based build management tool, like Gradle.
This approach pushes all the build logic into the developer's local environment. Now, developers on the
project have a build configuration they can run locally that is [hopefully] the same configuration executed on the
build server. The trade-off is that those local build configurations can get pretty complex, and they often rely on 
3rd party plugins to manage projects in other languages (e.g. building an NPM project with Gradle).

---

#### Problem 4: Deployment Orchestration

As was stated previously, almost all DevOps automation products are really build automation products. They are rebranded
CI tools. Sure, you can do deployments, but that's not really what they do. There are however, some exceptions, like Harness,
which is geared more towards deployment orchestration.

Taking into account that software systems must be both built and deployed, the lines can get blurred in the build configurations.
Not long ago, it wasn't uncommon to have deployment tasks baked into local build automation (ex. hot deploying to an application server).
The build server could then just invoke the deployment task during a deployment stage.

However, as the cloud, PaaS and containers have become more commonplace across the IT landscape, deployments are no longer
as trivial as they once were. Now, a software system's infrastructure isn't something provisioned in a backroom in some
unknown corner of a datacenter; it's defined as software alongside the rest of the system's software. And as such,
support for that infrastructure software and the associated system's deployments should be provided, both in terms
of local and remote deployments.

---

#### Problem 5: Disconnected Interfaces

Developers write/build/test/interact with code on their workstations. So, of course they want to stop what they are doing,
jump to a special Web portal and futz around with custom build automation just to get through their day ```**sarcasm**```.
The interface a developer has to their projects is their development environment, which typically consists of an 
Interactive Development Environment (IDE) and a command line shell. Yet another portal does not make developers more productive.
Bringing the tools developers need into their development environments does.

It's naive to think that a tool like a build server, which must be available at [nearly] all times, and which provides a 
service central to software development can or would exist without a Web interface. But it's also absurd to think that
it should be a time-consuming effort to navigate that interface or that it should even be the primary interface for developers.
One tool that seemingly understands this is Terraform Cloud, which provides a relatively simple web interface, but which
also integrates seamlessly with the Terraform CLI (Command Line Interface).

---

### Conclusions

There is no perfect way to do DevOps automation and there is no perfect automation tool to help you get there. There are some automation tools/services that
will get you up and running quickly, but they won't be easy to extend or come with a robust feature set. There are other automation tools/services
that are downright painful to get up and running but that you can customize to the moon. There are also a few in the middle. But
if you have to choose (and you kind of do), then a good argument can be made for selecting tooling that will get you up and
running quickly. After all, upfront value realized is better than the prospect of value on some future date with a lot of upfront pain
trying to get there.

Another major factor to consider is the ability to provide developer self-service. Developer self-service is inhibited, in no small part, through
problems 2-5 (above). Although those problems are not insurmountable, they can be costly.

Many developers and organizations spend a significant amount of time trying to provide some level of self-service in their quest for DevOps nirvana. 
In some cases, and after a significant investment, the juice stops being worth the squeeze and they end up with some custom portal to click around. 
But for the few who do have real self-service, they didn't get it from a Web interface, and they probably didn't get it 
from a packaged tool (with the possible exception of Terraform Cloud). 

The thing about service is that it is something provided to the customer. Developer self-service must be provided where the developer is. 
If what you want is self-service and all you have is a portal with a Web services API (as is often the case), then 
someone (a team? a developer?) has to create that self-service functionality... 
at least until DevOps automation tool providers get serious about baking more robust self-service support into their products.
