---
title: "Testing Terraform For More Than Just Compliance"
date: 2021-04-20T10:00:00-04:00
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

We discussed automated [Terraform testing using Spock](/blog/spock-terraform/). We discussed 
[zero-downtime deployments with autoscaling groups](/blog/asg-zero-downtime-tf/). Now, we're going to 
discuss implementing automated Terraform tests with Spock that do more than just test for compliance. That's right!
We're going to implement tests that verify the functionality! Because let's be honest,
if your infrastructure is compliant, but it doesn't doesn't work as expected then what do you really have? A: Compliant Junk.

![Groovy Language](/assets/images/posts/testing_with_spock/groovy.png)

_<small>Credit: The Apache Groovy programming language</small>_

## Setting the Stage ##

Before we get started, it might be beneficial to read the previous posts that this post builds upon.
* [Zero-Downtime AWS Autoscaling Groups with Terraform](/blog/asg-zero-downtime-tf/)
* [Testing Terraform With Spock](/blog/spock-terraform/)

We'll also be borrowing some of the codebase from the [Adding Capacity Matching to Terraform ASG Updates](/blog/asg-capacity-matching/) post.

Or... you could just checkout [the complete working example for this post on GitHub](https://github.com/BluFlameTech/examples/tree/main/tf-maven/zero-dt-asg-matching).

To get started, let's first set up our pom.xml as follows.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <artifactId>zero-dt-asg-matching</artifactId>
  <groupId>com.bluflametech.example</groupId>
  <version>0.1</version>
  <properties>
    <maven.compiler.source>15</maven.compiler.source>
    <maven.compiler.target>15</maven.compiler.target>
    <tf.maven.version>0.12</tf.maven.version>
    <test.types>syncinfra</test.types>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.codehaus.groovy</groupId>
      <artifactId>groovy</artifactId>
      <version>3.0.4</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.codehaus.groovy</groupId>
      <artifactId>groovy-ant</artifactId>
      <version>3.0.4</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.codenarc</groupId>
      <artifactId>CodeNarc</artifactId>
      <version>2.0.0</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.spockframework</groupId>
      <artifactId>spock-core</artifactId>
      <version>2.0-M2-groovy-3.0</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-simple</artifactId>
      <version>1.7.30</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk</artifactId>
      <version>1.11.959</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.deliveredtechnologies</groupId>
      <artifactId>tf-cmd-api</artifactId>
      <version>${tf.maven.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <!-- for executing terraform commands (e.g. tf:apply) -->
        <groupId>com.deliveredtechnologies</groupId>
        <artifactId>tf-maven-plugin</artifactId>
        <version>${tf.maven.version}</version>
      </plugin>
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
        <artifactId>maven-compiler-plugin</artifactId>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.0.0-M4</version>
        <configuration>
          <includes>
            <include>**/*Spec.java</include>
          </includes>
          <systemPropertyVariables>
            <testTypes>${test.types}</testTypes>
          </systemPropertyVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

Let's also assume that we're using [the following configurations for Spock and CodeNarc](https://github.com/BluFlameTech/examples/tree/main/tf-maven/zero-dt-asg-matching/src/test/resources/), respectively.

And just like in the [Testing Terraform With Spock](/blog/spock-terraform/), we will use the Annotations we defined as follows.

```groovy
package com.bluflametech.test

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
@interface DatabaseTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface SyncInfraTest { }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface AsyncInfraTest { }
```

Finally, we also have a TerraformSpecification defined, which our Terraform Spock Specifications will extend.

```groovy
/**
 * Spock Specification for testing Terraform modules.
 */
abstract class TerraformSpecification extends Specification {

  /**
   * Provides protected access to Terraform operations as follows.
   *
   * terraform.init()
   * terraform.apply([tfVars: tfvars])
   * terraform.output()
   * terraform.destroy()
   *
   * Notes on return values:
   *
   * terraform.apply([tfVars: tfvars]) returns a Json object containing the terraform output
   * terraform.init() returns the terraform object - so, terraform.init().apply([tfVars: tfvars]) is possible
   */
  private static Map<String, String> lastAppliedTfVars

  protected Map<String, Closure<?>> terraform

  {
    String tfRootDirUnderTest = tfRootDir
    def apply = new TerraformApply(tfRootDirUnderTest)
    def destroy = new TerraformDestroy(tfRootDirUnderTest)
    def output = new TerraformOutput(tfRootDirUnderTest)
    def init = new TerraformInit(tfRootDirUnderTest)

    terraform = [
            init: {
              init.execute([:])
              terraform
            },
            apply: { props ->
              setLastAppliedTfVars(props.clone()?.tfVars)
              apply.execute(props)
              terraform.output()
            },
            output: {
              JsonSlurper slurper = new JsonSlurper()
              slurper.parseText(output.execute([:]))
            },
            destroy: { destroy.execute([tfVars: getLastAppliedTfVars()]) }
    ]
  }

  protected synchronized static setLastAppliedTfVars(Map<String, String> tfVars) {
    lastAppliedTfVars = tfVars
  }

  protected synchronized static Map<String, String> getLastAppliedTfVars() {
    lastAppliedTfVars.clone() as Map<String, String>
  }

  /**
   * Determines the corresponding Terraform root module based on the name of the Specification.
   *
   * Specification class names are assumed to be named by convention as shown by example below.
   *
   * Specification name MyTfRootModule maps to the Terraform root module directory my_tf_root_module.
   *
   * The location of the Terraform root module can be either src/main/tf or src/main/tf-examples, with tf-examples
   * taking precedence.
   *
   * @return the String path of the matching Terraform root module under test
   */
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

You'll notice that some adjustments were made since the [Testing Terraform With Spock](/blog/spock-terraform/) post.
Destroy is no longer an implicit operation and tfVars can be carried over from one test to another. This
allows us to chain tests together in sequence, which would normally be considered a poor practice. However, infrastructure 
testing plays by a slightly different set of rules. And chaining tests together allows us to isolate lifecycle events (like updates)
in different tests. More on that below.

Other interesting things of note are that Terraform configuration directories are resolved based on convention and
destroy implicitly reuses the same tfVars that were passed into the prior apply. So, there's a lot less parameter juggling.

#### The Updated Terraform Configuration 

A few updates were made to the Terraform code from the [Adding Capacity Matching to Terraform ASG Updates](/blog/asg-capacity-matching/) post.
In this post's Terraform code, we are using a simple Express server that is created in the userdata script. The Express
server serves up the _message_ supplied in the tfvars. This allows us to easily distinguish beteween the initially deployed
autoscaling group and the new autoscaling group that is deployed on update. It also means we don't need an S3 bucket.

## Our First Test ##

Our first test is really just going to test that we can deploy an auto-scaled web application to the desired capacity and 
that it works (i.e. it serves up the desired HTTP response).

```groovy
class ZeroDtAsgMatchingSpec extends TerraformSpecification {

  private String[] messages = ['first', 'second']

  def 'asg for web app with classic elb provisioned'() {

    given:

    def region = System.getProperty('region', 'us-east-1')
    AmazonElasticLoadBalancing elb = AmazonElasticLoadBalancingClientBuilder.standard().withRegion(region).build()

    def tfVars = [
            region       : region,
            vpc_id       : System.properties['vpc_id'],
            app_name     : 'test',
            message      : messages[0],
            public_domain: System.properties['public_domain'],
            min_size     : '2'
    ]

    when: 'autoscaling group web app server with elb is provisioned'

    def output = terraform.init().apply([tfVars: tfVars])

    then:

    output.elb.value.dns ==~ /\b${tfVars.app_name}-\b.{3,}\b.${tfVars.public_domain}\b/

    LoadBalancerDescription awsElbDescription = elb.describeLoadBalancers({
      def describeRequest = new DescribeLoadBalancersRequest()
      describeRequest.withLoadBalancerNames(output.elb.value.name)
    }.call()).loadBalancerDescriptions[0]

    awsElbDescription.instances.size() == (tfVars.min_size as Integer)
    awsElbDescription.listenerDescriptions.size() == 1
    awsElbDescription.listenerDescriptions[0].listener.loadBalancerPort == 443
    awsElbDescription.listenerDescriptions[0].listener.instancePort == 80

    waitForInstancesInService(elb, DescribeInstanceHealthRequest
            .getDeclaredConstructor()
            .newInstance()
            .withInstances(awsElbDescription.instances)
            .withLoadBalancerName(output.elb.value.name))

    httpGet("https://${output.elb.value.dns}").body() == tfVars.message
    
    cleanup:
    
    terraform.destroy()
  }
}
```

In this test, the autoscaling group is provisioned with a classic ELB. Then, we do some basic compliance checks:
we check the ELB's DNS, we check the number of instances provisioned and we check the listener configuration.
Finally, we check that it does what it's supposed to do: create the proper response when we make a HTTP request.
And after all of that is done, we tear it down using ```terraform.destroy()```.

To faciliate this test, we added two methods: waitForInstancesInService() and httpGet(). We define them below.

```groovy
private static boolean waitForInstancesInService(AmazonElasticLoadBalancing elb, DescribeInstanceHealthRequest healthRequest) {
    for (int iteration: (1..10)) { //checks for all instances to be 'InService' for 10 iterations
        def instanceHealth = elb.describeInstanceHealth(healthRequest)
        if (instanceHealth.instanceStates.stream().allMatch{instance -> instance.state == 'InService'}) {
            return true
        }
        sleep(30000) //sleeps for 30sec after each check - so, 5min w/10 iterations
    }
    false
}

private static HttpResponse<String> httpGet(String uri) {
    HttpClient client = HttpClient.newHttpClient()
    HttpRequest request = HttpRequest.newBuilder(URI.create(uri)).GET().build()
    client.send(request, HttpResponse.BodyHandlers.ofString())
}
```

## Our Second Test ##

Our second test has most of the meat. Our second test tests that when an update happens, there is no downtime. It also tests that once 
the update is complete, the updated response is returned when we make an HTTP request. That seems simple enough, but there
are a few gotchas:
1. How can we ensure the second test is tethered to the first test?
2. How can we check for downtime while an update is in progress?

The answer to the first question is Spock's built-in @Stepwise annotation. Annotating a Specification with
@Stepwise ensures that tests are executed in the order in which they are declared, one right after the other. And as far
as making state in the first test available to the second test, well, Terraform kind of already does that. But to help
facilitate that shared state, we have ```lastAppliedTfVars``` and ```terraform.output()```, both of which are baked
into TerraformSpecification.

The answer to the second question is asynchronous execution. In Groovy, that means spawning a thread. We have
two choices as to what should be in the spawned thread: the Terraform update or checking for continuous uptime. Since
tests are really the star of the show, let's put the Terraform update in the spawned thread and we'll keep checking for
continuous uptime in the main thread... and at some point before the end of the test, they'll join.

Without further delay, here's the code for the second test.

```groovy
def 'asg is updated without downtime'() {

    given:

    def tfVars = lastAppliedTfVars

    when: 'autoscaling group is updated'

    tfVars.message = messages[1]
    def output = terraform.output()
    def job = AsyncJob.create({ terraform.init().apply([tfVars: tfVars]) })

    then: 'original asg is replaced under the load balancer with zero downtime'

    while (!job.done) { //continuously test that HTTP requests are working and they are returning the correct values
        httpGet("https://${output.elb.value.dns}").body() ==~ /(\b${messages[0]}\b|\b${messages[1]}\b)/
    }

    //now that the update is complete, it should return only the updated response
    httpGet("https://${output.elb.value.dns}").body() == messages[1]

    cleanup:

    terraform.destroy()
}
```

Pretty simple, huh? Don't let it's terseness fool you; it does a lot.

But there are some holes we still need to fill in. like, "What is that AsyncJob class?" And "Didn't the first 
test call a ```terraform.destroy()``` at the end?" Fair enough. Here is the AsyncJob class definition.

```groovy
class AsyncJob<T> {
    private CompletionService<T> completionService
    private Future<T> future

    static <U> AsyncJob create(Callable<U> callable) {
        new AsyncJob(callable)
    }

    private AsyncJob(Callable<T> callable) {
        Executor executor = Executors.newSingleThreadExecutor()
        completionService = new ExecutorCompletionService<>(executor)
        future = completionService.submit(callable)
    }

    boolean isDone() {
        future.done
    }

    T getResult() {
        if (!done) {
            return null
        }
        completionService.take().get()
    }
}
```

As for the ```terraform.destroy()``` at the end of the first test, yea, you'll have to remove that. Actually,
you'll want to remove the entire cleanup section from that test.

Once that's done, you should be able to drop to the command line at the root of your project (where the pom.xml is located) and
type ```mvn clean test -Dvpc_id="{insert vpc id here}" -Dpublic_domain="{insert your registered domain here}"``` to run the tests.

Finally, a few things of note:
1. The VPC you use must have public and private subnets, tagged with tier=public and tier=private, respectively
2. The domain you specify must be a public domain with a *.{domain} certificate registered in ACM

Of course, that's just for this example. Feel free to create your own.

## Summary ##

* Compiance great, but working is better (compliance + working = best)
* Spock has a ton of tricks to facilitate both compliance testing and to help validate the functionality of Terraform configurations
* Sometimes you have to think a little outside the box when creating infrastructure tests
* The complete working example project associated with this post is located in [GitHub](https://github.com/BluFlameTech/examples/tree/main/tf-maven/zero-dt-asg-matching). 
