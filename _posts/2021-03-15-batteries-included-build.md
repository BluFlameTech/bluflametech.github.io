---
title: "Batteries Included Build Automation"
date: 2021-03-14T17:05:00-04:00
header:
  teaser: /assets/images/posts/batteries_included_build/battery.png
categories:
- blog 
tags:
- devops
- build
- automation
- solutions
---

Clone repository, run local build... build failure. It works on the build server. It also works on Grant's workstation. 
But it doesn't work for you. Does this sound familiar? If so, then you may be a victim of Irritating Build Syndrome (IBS). 
Fortunately, there is a cure.

![Batteries Included](/assets/images/posts/batteries_included_build/battery.png)

_<small>Image by [Leyn](https://www.istockphoto.com/portfolio/Leyn?mediatype=illustration) on [iStock](https://www.istockphoto.com) by Getty Images</small>_

Nobody wants to chase down umpteen build dependencies just to get to the point where they can start work. It's not just
a problem for new team members. Anyone who has been in the software development game for any period of time has surely
experienced a nero (near zero progress) day due to a broken local build environment.

### A Brief History of [Some] Build Automation Tools

From Make to Ant to tools like Maven, Gradle and NPM, build automation has had a long evolutionary history. Decades ago,
Ant was the cool new built tool on the block. Those who built Windows applications (including this author), remember
the joys of trying to manage DLL Hell with InstallShield, too. At the time, most builds shared basic commonalities, but
almost all of them were lovingly handcrafted. A good build engineer was worth his or her weight in gold because they
were the foundation of the team's ability to produce working software. 

Five years after Ant was released, Maven was released, and with it came a giant leap forward for build automation. Like Make,
Ant was basically build automation through scripting, albeit scripting that hooked into Java code. Maven, on the other
hand brought with it [project management and dependency management](https://maven.apache.org/what-is-maven.html) in 
addition to build automation. Maven also heavily favored
the use of conventions. So, unlike Ant where "ant" could mean different things to different projects based on whatever
what was in the build.xml (which could be anything), with Maven "mvn install" was 
[a phase](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html) that had meaning attached to it. 

Maven's dependency management is still the de-facto standard for Java projects and since its introduction, similar 
dependency management functionality has been introduced for other languages and platforms, like 
[NuGet for .Net](https://docs.microsoft.com/en-us/nuget/what-is-nuget). Building
upon Maven's dependency management functionality, and borrowing similar project management functionality, Gradle was released
2 years after Maven's initial release. [Gradle](https://gradle.org/maven-vs-gradle/) combined [Groovy](https://groovy-lang.org/) 
scripting with a convention-based project approach to build automation. Gradle gained more prominence in 2014,
with [the release of Google's Android Studio](https://venturebeat.com/2014/12/08/google-releases-android-studio-1-0-the-first-stable-version-of-its-ide/) 
with [Gradle-based build support](https://developer.android.com/studio/build).

_<small>It should be noted that in terms of dependency management, Maven was not the first. 
For example, the introduction of PyPI predates Maven by 2 years.</small>_

A lot of Java build automation tooling was discussed above and for a good reason. Although, Java doesn't enjoy the language
popularity it once did, Java and JVM-based languages are still quite [popular](https://www.tiobe.com/tiobe-index/). 
Java has also been around since 1995. So, it's a great example to illustrate how build automation tools have evolved over 
the past 20+ years.

### Modern Challenges

With all the advances in build automation tooling, one might think that the problems build automation faced in the past 
have long been solved. Unfortunately, builds still break, dependencies still go missing and often times a working 
and productive development environment can take days to set up. Brittle builds are still a reality that are best
discussed using an example.

> What we need is a way to ensure all the conditions necessary for a successful build can be met with minimal effort.

### The Example Project

Let's assume we have a relatively small system consisting of a REST API written in Java, a Web front-end written in ReactJS, a
command line interface (CLI) written in Python and a couple of Terraform configurations that define the infrastructure
needed to host the REST API and the Web front-end, respectively. In order to build this system in its entirety, a compatible
version of Java is needed, a compatible version of Python is needed, and a compatible version of Node/NPM is needed. Additionally,
the necessary Java build automation tool must be installed (probably Maven or Gradle), all code library dependencies 
must be available. That's a lot of conditions that must be met prior to being able to build the system. What we need is 
a way to ensure all the conditions necessary for a successful build can be met with minimal effort.

```yaml
java-rest-api:
   build.gradle
python-cli:
   setup.py
   requirements.txt
js-frontend:
   package.json
tf-api-host:
   terrafile
tf-frontend-host:
   terrafile
```

_<small>Hierarchical Representation of Project</small>_

#### Developer Virtual Workstations

Providing every developer a virtual workstation that's 100% bootstrapped with all the tools and configuration they need to successfully
checkout and build the entire system is one way to both speed up onboarding and guard against a mucked up development environment.
Tools like [Vagrant](https://www.vagrantup.com/) make this option both viable and streamlined.

Using this approach, an image can either be packaged/baked and published or configured using a published Vagrantfile. Additionally,
developers can personalize their environment through extensions or by modifying their Vagrantfile to suit their
preferences. If for some reason a _"doesn't build on my workstation"_ affliction occurs, a working environment is just a new
```vagrant up``` away.

A similar option is the use of VMs in the cloud. Just be aware of network connectivity and lag. Waiting 5 seconds for an image to load 
in a browser is acceptable. Waiting 5 seconds for a keystroke to register on the command line is not.

#### Single Point of Entry Build Automation

The drawback to resolving environment dependencies solely with developer virtual workstations is that the build server and the 
developer virtual workstations must be kept in sync, which 
may or may not be a headache. But there are few things more frustrating than having a working build on your developer 
workstation, only to find out that it doesn't work on the build server because it's got a different version of some dependency installed.
Additionally, if the development environment is virtual then it may not be as performant as it would be on [bare metal](https://www.pcmag.com/encyclopedia/term/bare-metal).

Having a single point of entry build automation solution could be for a bare metal developer workstation or it could be 
used with a developer virtual workstation. To better understand single point of entry build automation, let's first 
consider how a build might work assuming any working development environment.

The java-rest-api project, the js-frontend project, the python-cli project, and the Terraform projects must all be 
built independently. That's 5 different build entry points for a single system. Then, there is still the question of
"Which dependencies are installed on the build server?" 

What if your version of NodeJS is different than the build server's version of NodeJS?
What if your version of Terraform is different than the build server's version of Terraform? What about your version of 
Maven or Gradle or Python?

##### Ex. Gradle as the Single Point of Entry

Both Maven and Gradle support the concept of multi-project builds. Consider the updated project structure below.

```yaml
build.gradle
settings.gradle
gradlew
java-rest-api:
  build.gradle
python-cli:
  build.gradle
  setup.py
  requirements.txt
js-frontend:
  build.gradle
  package.json
tf-api-host:
  build.gradle
  terrafile
tf-frontend-host:
  build.gradle
  terrafile
```

What if we could create a multi-project build like the one above and build the whole thing using Gradle from the parent
project? What if we could also ensure that the versions of Gradle, Node, Terraform and Python used were always consistent?
With [Gradle multi-project builds](https://docs.gradle.org/current/userguide/multi_project_builds.html), 
a single command ```gradlew build``` at the root can build all the sub-projects. What _build_ means can also be 
adjusted to suit the type of project. 

As far as project and language support, there is a [Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html), 
that ensures all builds use the same version of Gradle. Gradle also has an [integration with Node & NPM](https://github.com/node-gradle/gradle-node-plugin) that will ensure
specific versions of Node & NPM are used. Similarly, there is also a [Gradle plugin for building Python](https://github.com/linkedin/pygradle).

There is not, however, a Gradle plugin for building Terraform. But there is a [Maven plugin for Terraform](https://github.com/deliveredtechnologies/terraform-maven),
which also publishes a [Java API for Terraform](https://search.maven.org/artifact/com.deliveredtechnologies/tf-cmd-api/0.12/jar) 
that could be used with Gradle.

By going down this route, there is still an external dependency on Java (Gradle, after all, is a Java-based tool). Gradle can, however,
validate that the correct version of Java is available as part of the build. And everything else (what software,
which versions, etc.) is dependent upon the build configuration stored with the project, not the developer's workstation and not 
the build server. The build configuration is checked into source control with the project. Where the source code goes, it goes. 

### Summary

Few things suck the productivity (and joy) out of software development like brittle build automation. It's in everyone's
interest to ensure that building the system from top to bottom can be done quickly, easily and with as few external software
dependencies as possible. One way to eliminate the burden of external software dependencies and to ensure a complete and functional
development environment is to automate it through something like Vagrant.

However, an automated development environment doesn't eliminate the need for a "batteries included" build. Multi-project
build organizations, and the appropriate use of wrappers and plugins can provide single point of entry build automation that can
bootstrap its own software dependencies. Being able to clone a repo consisting of multiple projects, languages and technologies,
and with a single command, reliably them all is both possible and reasonable. 

Don't you think the savings you can gain in time and frustration is well worth little effort required to make it happen?
