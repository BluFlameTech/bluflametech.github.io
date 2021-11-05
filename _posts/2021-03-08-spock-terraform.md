---
title: "Testing Terraform With Spock"
date: 2021-03-08T16:38:00-05:00
header:
  teaser: /assets/images/posts/testing_with_spock/groovy.png
categories:
- blog 
tags:
- devops
- testing
- infrastructure
- java
- terraform
- groovy
- automation
- solutions
---

For many modern programming languages, various levels of automated testing have been around for years. However, for Infrastructure
as Code (IaC), and specifically Terraform, the landscape is much less mature. Wouldn't it be great if we could bring both
Java tests and Terraform tests together under the same roof... er... project using a simple and arguably elegant BDD-style
testing framework. We can! Enter Spock.

![Groovy Language](/assets/images/posts/testing_with_spock/groovy.png)

_<small>Credit: [The Apache Groovy programming language](https://groovy-lang.org/)</small>_

### The Terraform Testing Landscape

The available testing tools for Terraform mainly consist of 

* [TerraTest](https://github.com/gruntwork-io/terratest)
* [Kitchen-Terraform](https://newcontext-oss.github.io/kitchen-terraform/) and [AWSpec](https://github.com/k1LoW/awspec) 
_(if you are using Terraform with AWS)_
* [InSpec](https://community.chef.io/tools/chef-inspec/). 
* [ServerSpec](https://serverspec.org/)
* [Terraform's Acceptance Testing Framework](https://github.com/hashicorp/terraform/blob/main/.github/CONTRIBUTING.md#acceptance-tests-testing-interactions-with-external-services)

Like Terraform itself, each has their own strengths and weaknesses. But none of them, except for TerraTest are geared towards validating that 
the infrastructure actually works. The other tools listed are really just compliance testing tools, which is great. Compliance
tests are easier and faster to evaluate and they do provide some value. But compliance and $2.00 will get you [a cup of 
coffee at StarBucks](https://www.fastfoodmenuprices.com/starbucks-prices/) if your infrastructure doesn't actually work.

Other issues with the above infrastructure test frameworks include poor langauge support for [the most widely used 
software development languages](https://www.tiobe.com/tiobe-index/) and their bespoke configurations. As much as 
heterogeneous programming language environments are touted, most large IT shops primarily support 1 or 2 development 
platforms consisting of .Net, Java and/or some flavor of JavaScript. Some traditional Ops engineers staff will likely
have experience with Bash and/or Python (and sometimes Ruby) and that's generally the extent.

_<small>The point of the above statements are not meant to poo-poo developers or ops traditionalists. Nor are they meant as a snub
to all the other great programming languages that were not mentioned. Rather, they are intended to frame the
available Terraform infrastructure testing options in the context of present day IT organization realities.</small>_

#### What About Cross-Functional Teams?

Just because a team consists of team members who can jockey between different langauges, tech stacks and competencies, it
doesn't mean the organization will let them _(Boo!)_. And even if the organization will let them , that doesn't mean language X or
tech stack Y will enjoy support within the organization. And even if those barriers are surmountable, the pain
of segmenting infrastructure tests and application tests is real. Different tools, each having different configurations 
to run different tests for the same logical piece of functionality is a drag factor rooted in complexity.

### The Spock Testing Framework

Our tech stack consists of different langauges and tech stacks: JavaScript, Java, Python, Terraform, etc. Different 
languages and tech stacks means more complexity. However, it's complexity that must be offset by value. And we continuously
look for ways to streamline it. One of those ways is through a single, batteries included build tool _(to be covered
in a future post)_. Another way is by streamlining (and combining) application and infrastructure tests. We use Spock to 
do that within our Java projects.

#### Why Spock?

[Spock](https://spockframework.org/spock/docs/1.3/index.html) is a [Groovy](https://groovy-lang.org/) BDD-style testing framework based on JUnit.
Spock gives you all the great stuff that JUnit offers and all the stuff that's commonly added onto JUnit (like interactions with mocks) in
an all-in-one streamlined testing framework. The Groovy-ness of Spock also allows for some gray-box testing, terseness of syntax not
afforded by Java, near 100% compatibility with Java, dynamic types and meta-programming (which should be used sparingly, 
but which can also be REALLY useful sometimes).

An entire post could be filled with why Spock is such a great (and easy to use) testing framework. But that is not this post.
So, without much further delay, let's discuss how we use it to test Terraform.

### Project Setup

At first, we decided on using Gradle for project-based build automation and dependency management. But a few things changed our minds
and nudged us toward Maven.

1. Maven has great support for [delegation to NPM](https://github.com/eirslett/frontend-maven-plugin) and other languages.
2. There is a [Maven plugin for Terraform](https://github.com/deliveredtechnologies/terraform-maven).

The initial thinking was that there was no way we wanted to do XML programming of any way, shape or form. And that's kind
of what you take on with Maven POMs. But as we progressed, the benefits of using Maven seemed to outweigh the effort required to entend
Gradle. So, now we're using Maven. **Less Time Spent on Scaffolding = More Time Spent Creating Value**

There's already some great posts out there about using [Terraform with Maven](https://medium.com/deliveredtechnologies/unit-testing-terraform-e592a5c3777f)
and about using [Groovy to test Terraform](https://medium.com/deliveredtechnologies/making-terraform-testing-groovy-6a9278bdce1).
This post won't regurgitate that content. Instead, we'll now demonstrate how we test Java and Terraform together using Spock
with Groovy.

#### Configuring Maven

First, add the tf-cmd-api, groovy, spock and aws-java-sdk dependencies to the pom.xml. We'll also add the reflections dependency for a little
configuration magic, which will be discussed shortly.

```xml
<dependency>
  <groupId>com.deliveredtechnologies</groupId>
  <artifactId>tf-cmd-api</artifactId>
  <version>${tf-maven-version}</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.codehaus.groovy</groupId>
  <artifactId>groovy</artifactId>
  <version>3.0.4</version>
</dependency>
<dependency>
  <groupId>org.reflections</groupId>
  <artifactId>reflections</artifactId>
<version>0.9.12</version>
</dependency>
<dependency>
  <groupId>org.spockframework</groupId>
  <artifactId>spock-core</artifactId>
  <version>2.0-M2-groovy-3.0</version>
<scope>test</scope>
</dependency>
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-java-sdk</artifactId>
  <version>1.11.838</version>
  <scope>test</scope>
</dependency>
```

Next, to facilitate the execution of Terraform operations using Maven, let's also add in the tf-maven-plugin configuration as follows.

```xml
<plugin>
  <groupId>com.deliveredtechnologies</groupId>
  <artifactId>tf-maven-plugin</artifactId>
  <version>${tf-maven-version}</version>
</plugin>
```

Finally, we need to add the Gmavenplus plugin configuration and the Surefire plugin configuration so that our Groovy code is converted
into Java and so that Spock Specifications _(Spock's name for test suites)_ are discovered by Surefire, respectively.

```xml
<plugin>
  <groupId>org.codehaus.gmavenplus</groupId>
  <artifactId>gmavenplus-plugin</artifactId>
  <version>1.9.0</version>
  <executions>
    <execution>
      <goals>
        <goal>addTestSources</goal>
        <goal>compileTests</goal>
      </goals>
    </execution>
  </executions>
</plugin>
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>3.0.0-M4</version>
  <configuration>
    <includes>
      <include>**/*Spec.java</include>
    </includes>
  </configuration>
</plugin>
```

#### Configuring Spock

It's great to have all your tests for your project in one place. But it's not so great if all those tests run every single time.
Infrastructure tests take time to run - a lot longer than Java tests. So, you probably don't want them running every time you
run your Java tests. Fortunately, Spock has a very simple mechanism that allows us to run tests of specific types at different times
or all-together.

Our basic approach is this:

1. Identify the types of tests you want to support
2. Create annotations for each of those test types
3. Configure Spock to select the correct test types based on defaults, properties and/or command line arguments

Let's assume the project holds different types of automated tests: unit tests, integration tests, synchronous infrastructure tests and 
asynchronous infrastructure tests. Unit tests should be the default type of tests run - they run the fastest and more often than
not, unit tests are what we want to validate on local builds. Then we have integration tests - tests that validate the integration between
the current project and other stuff, synhcronous infrastructure tests - infrastructure tests we can get feedback on relatively quickly and 
finally, asynchronous infrastructure tests - infrastructure tests that have a relatively long feedback cycle (e.g. testing S3 replication).

##### Creating The Annotations

Add a single Annotations.groovy file in a test package as follows.

```groovy
package com.bluflametech.test //your package name goes here

import java.lang.annotation.ElementType
import java.lang.annotation.Retention
import java.lang.annotation.RetentionPolicy
import java.lang.annotation.Target

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface AlwaysTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface UnitTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface IntegrationTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface SyncInfraTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface AsyncInfraTest { }
```

##### Configuring Spock

Add a single SpockConfig.groovy file in the test/resources directory as follows.

```groovy
import com.bluflametech.test.AlwaysTest
import com.bluflametech.test.UnitTest
import org.reflections.Reflections

import java.lang.annotation.Annotation
import java.util.stream.Collectors

def testAnnotations = (new Reflections('com.bluflametech.test'))
    .getSubTypesOf(Annotation)
    .stream()
        .filter({ clazz -> !clazz.simpleName.toLowerCase().startsWith('always')})
        .collect(
            Collectors.toMap(
                {Class<?> clazz -> clazz.simpleName.toLowerCase().dropRight(4)},
                {Class<?> clazz -> clazz}))

def testTypes = System.getProperty('testTypes')?.split(',\\s*')?.collect {testAnnotations[it.toLowerCase()]}

runner {

  testTypes?.each { testType ->
      include.annotations << testType
  }
  include.annotations << AlwaysTest
  testTypes ?: include.annotations << UnitTest
}
```

Deconstructing the code, we can see that it does the following.

```def testAnnotations = ...``` creates a Map of all annotations in the specified package. Assuming the above code that
created the annotations, the resulting map is as follows.

```groovy
[
  unit: UnitTest,
  integration: IntegrationTest,
  syncinfra: SyncInfra,
  asyncinfra: AsyncInfra
]
```

```def testTypes = ...``` gets the _testTypes_ system property, which can be passed in like ```mvn test -DtestTypes=unit,integration```
from the command line.

The _runner_ closure configures the Spock runner, which will add each annotation associated with its key value (above)
specified in the testTypes parameter. If no testTypes are specified, Spock specificiations annotated with ```@UnitTest```
are run by default. And anything annotated with ```@AlwaysTest``` is always run as part of a test execution. This allows
stuff like linting to be configured in terms of Spock specifications.

### Adding Some Syntactic Sugar

All the basics for wiring up Terraform tests with Spock are included in the tf-cmd-api dependency that was added above.
However, that library lacks the terseness and the elegance that Spock provides. In short, it's not very Groovy. Fortunately,
Groovy comes with some really powerful features that can help out.

First, let's talk about what's missing.

**1. Destroy on cleanup**

A typical Terraform test has 3 parts: 1. provision infrastructure using Terraform 2. evaluate provisioned infrastructure 3. tear down provisioned
infrastructure. The infrastructure for each test might be a little different, how it's evaluated will be different, but the need to destroy
the infrastructure is a constant. Why not just make it happen without having to add extra code?

**2. Instantiate Terraform operations by default**

The tf-cmd-api library requires Terraform operations to be instantiated as follows.

```groovy
TerraformInit init = new TerraformInit(TF_ROOT_DIR)
TerraformInit apply = new TerraformInit(TF_ROOT_DIR)
```

Wouldn't it be nice if they were just wired up and made available?

**3. Intuitive usage of Terraform operations**

Without extension, the above Terraform operations would be used as follows.

```groovy
init.execute([:])
apply.execute([tfVars: props])
```

That's not bad. But we can do better.

```groovy
terraform.init()
terraform.apply([tfVars: props])
```

Now, that looks more intuitive. Let's make it happen!

#### A Terraform Specification for Spock

Test suites in Spock are Specifications. They are created by extending the Spock Specification class. So, why not have
a TerraformSpecification class that wires up some Terraform functionality?

```groovy
package com.bluflametech.test

import com.deliveredtechnologies.terraform.TerraformUtils
import com.deliveredtechnologies.terraform.api.TerraformApply
import com.deliveredtechnologies.terraform.api.TerraformInit
import com.deliveredtechnologies.terraform.api.TerraformOutput
import com.deliveredtechnologies.terraform.api.TerraformPlan
import groovy.json.JsonSlurper
import spock.lang.Specification

abstract class TerraformSpecification extends Specification {

  private String tfRootDirUnderTest
  protected Map<String, Closure<?>> terraform

  {
    String destroyPlanFile = 'destroy.tfplan'
    tfRootDirUnderTest = tfRootDir
    def apply = new TerraformApply(tfRootDirUnderTest)
    def plan = new TerraformPlan(tfRootDirUnderTest)
    def output = new TerraformOutput(tfRootDirUnderTest)
    def init = new TerraformInit(tfRootDirUnderTest)

    terraform = [
        init: { init.execute([:]) },
        apply: { props ->
          Properties planProps = props.clone() as Properties
          planProps << [destroyPlan: 'true']
          planProps << [planOutputFile: destroyPlanFile]
          apply.execute(props)
          plan.execute(planProps)
          JsonSlurper slurper = new JsonSlurper()
          slurper.parseText(output.execute([:]))
        },
        destroy: { apply.execute([plan: destroyPlanFile]) }
    ]
    
    removePlans()
  }

  def cleanup() {
    terraform.destroy()
    removePlans()
  }

  private void removePlans() {
    File dir = TerraformUtils.getTerraformRootModuleDir(tfRootDirUnderTest).toFile()
    File[] planFiles = dir.listFiles((FilenameFilter) { foo, name -> (name as String).endsWith('.tfplan') })
    planFiles.each { file -> file.delete() }
  }
  
  private String getTfRootDir() {
    String className = this.getClass().simpleName
    String stackName = className[0].toLowerCase() +
        className.dropRight(4)[1..-1].replaceAll('[A-Z]') { matcher ->
          "_${matcher[0].toLowerCase()}"
        }
    File example = new File("src/main/tf-examples/${stackName}")
    if (example.exists() && example.directory) {
      return "tf-examples/${stackName}"
    }

    stackName
  }
}
```

Let's start at the bottom. ```getTfRootDir()``` resolves the Terraform root module directory based on the name of the 
specification. By convention, specifications end with the suffix _Spec_. ```getTfRootDir()``` amends that convention by
matching everything before that suffix with a root module directory.

Example:

| Class Name            | Root Module Directory |
|-----------------------|-----------------------|
| BucketSpec            | bucket                |
| BucketWithReplicaSpec | bucket_with_replica   |

The search order for the Terraform root module directories is first _src/main/tf-examples/<root module directory>_ and
then _src/main/tf/<root module directory>_.

Now, let's move to the top of that file where the protected _terraform_ instance variable is declared. This allows access
to Terraform commands in the spec like ```terraform.init()```. Notice that it's a map of closures - this is done so
that we can control how the commands are executed.

Moving down from there is the initialization block. Spock does not allow subclasses of its Specification class to
declare their own constructors. So, instead of a constructor, we use an initialization block. This is where we can load
up the ```terraform``` closure map with Terraform operations.

Since we expect the Terraform init operation to be called with no arguments, we supply an empty map to its call and wrap that
call in a no argument closure. Next, we define ```terraform.apply``` to be a Terraform apply operation, the creation of a
Terraform destroy plan (so that tfvars don't have to be supplied to tear down the infrastructure on cleanup) and a Json object
representation of ```terraform output``` as the return value. Finally,
we define the ```terraform.destroy``` operation to be a Terraform apply using the destroy plan that was created on ```terraform.apply```.

Just below the initialization block is the ```cleanup()``` method. The ```cleanup()``` method is a special method for Spock
that is executed following every test, whether the test succeeds or fails. It's a perfect place to destroy the Terraform
infrastructure that was provisioned during the test. Additionally, in the ```cleanup()``` method and in the initialization block,
we can guard against stale plans by ensuring that any plans are cleaned up. Thus keeping inline with a cardinal rule of 
testing: **Always clean up after yourself**.

### Writing a Spock Test for Terraform

Now that we've added our dependencies to Maven, configured Spock and added some syntactic sugar to make testing more 
[DRY](https://www.artima.com/articles/orthogonality-and-the-dry-principle), elegant and intuitive, let's take a look at what a (simple) Terraform test with Spock might look like.

```groovy
@SyncInfraTest
class BucketSpec extends TerraformSpecification {

  def "bucket tf module creates a non-kms encrypted private bucket"() {
    given:

    AmazonS3 s3 = AmazonS3ClientBuilder.standard().withRegion('us-east-1').build()

    Faker faker = new Faker(new Random(System.currentTimeMillis()))

    def tfvars = [
        region: 'us-east-1',
        bucket_name: "bftest-${faker.funnyName().name().toLowerCase().replaceAll('[.\']', '').replaceAll('[\\s]', '-')}" as String,
        product: 'bluflametech',
        env: 'shared'
    ]

    when:

    terraform.init()
    def output = terraform.apply([tfVars: tfvars])

    then:

    //validate outputs
    output.details.value.bucket_name == tfvars.bucket_name
    output.details.value.bucket_region == tfvars.region
    output.details.value.website_endpoint == 'none'
    output.details.value.kms_key_arn == 'none'

    //validate deployment assumptions in AWS
    s3.doesBucketExistV2(tfvars.bucket_name)
    with(s3.getBucketTaggingConfiguration(tfvars.bucket_name).tagSet) { bucket ->
      bucket.getTag('product') == tfvars.product
      bucket.getTag('env') == tfvars.env
    }

    GetPublicAccessBlockResult publicAccessBlockResult =
        s3.getPublicAccessBlock((new GetPublicAccessBlockRequest()).withBucketName(tfvars.bucket_name))
    publicAccessBlockResult.publicAccessBlockConfiguration.blockPublicAcls
    publicAccessBlockResult.publicAccessBlockConfiguration.blockPublicPolicy
    publicAccessBlockResult.publicAccessBlockConfiguration.ignorePublicAcls
    publicAccessBlockResult.publicAccessBlockConfiguration.restrictPublicBuckets
  }
}
```

Pretty cool, right? We think so.

However, you may have noticed that even after the (albeit brief) discussion on compliance testing above, the example directly
above is, in fact, a compliance test. The good news is that Spock, the Terraform Maven plugin and its libraries can do more - much more.

But that will have to wait for a future post.