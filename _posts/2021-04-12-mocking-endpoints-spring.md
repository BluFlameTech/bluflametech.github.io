---
title: "Mocking Service Endpoints (With Spring Boot)"
date: 2021-04-12T08:30:30-04:00
header:
  teaser: /assets/images/posts/mock_endpoints/SpringMocks2.jpg
categories:
  - blog
tags:
  - services
  - mocks
  - web
  - spring
---

Do you interact with external web services? Do you need to test in a controlled environment where those external 
services might not be available? If you answered "yes" then you want to have mock service endpoints. Let's talk about
how we can (easily) implement mock service endpoints using Spring Boot and WireMock.

![GitOps](/assets/images/posts/mock_endpoints/SpringMocks2.jpg)

_<small>Credit: Blu Flame Technologies</small>_

### Why Mock Service Endpoints? ###

We kind of gave away the punchline in the teaser above. :)

There are a couple of scenarios in which mock service endpoints might be beneficial.
* Your system requires a callback from an external system
* Your system makes a call to an external system

An example of a callback might be the use of an external identity provider (IdP). Sure you could disable your auth
workflow when testing. But don't you also want to verify that your auth workflow works as expected?

The same holds true for any external service that your system relies on for functionality. You want to make
sure that functionality works as expected and that the different paths of execution are evaluated under test. That's
exactly what service endpoint mocks do: they validate that your system works as expected with external services by simulating
those external services.

### Approaches for Implementing Mock Service Endpoints ###

#### 1. Mock Server ####

A mock server can be a physical or a virtual server. It can be run on the same machine or on a different machine than the
system under test. But the basic functionlity is the same. Mock servers provide endpoint simulations as a service that is 
decoupled from the system under test. Below we will discuss how to use the mock server WireMock.

#### 2. Mock Services ####

Like a mock server, mock services also provide endpoint simulations. However, unlike a mock server, mock services are 
contained within the system under test. One obvious benefit to using mock services is that outages and connectivity issues
are virtually a non-existent problem. Because the mock services are part of the system under test,
if the system being tested is available, its mock services are also available. Of course, some care needs to be taken to
ensure that the mock services aren't deployed (or enabled) inappropriately (i.e. in a live/customer facing environment).

### Configuring The WireMock Dependency ###

[WireMock](http://wiremock.org/docs/getting-started/) was chosen for the mock server mostly because it's easy to use and it's reasonably well documented. There are,
however a few hiccups. Most notably, it's not super straight forward how to run a WireMock server and a Spring Boot
application with an embedded servlet container side-by-side outside of tests. But since we don't really need to do that,
anyway, it really doesn't matter.

Configuring WireMock as a dependency couldn't be any easier. For Maven, just add the following dependency to the pom.xml
and WireMock is ready to be used inside your test suites.

```xml
    <dependency>
      <groupId>com.github.tomakehurst</groupId>
      <artifactId>wiremock-standalone</artifactId>
      <version>2.27.2</version>
      <scope>test</scope>
    </dependency>
```

### Using WireMock for Tests ###

Starting up a WireMock server in a test suite is pretty trivial. Using [Groovy](https://groovy-lang.org/),
it looks like this.

```groovy
WireMockServer mockServer = new WireMockServer()
mockServer.start()
```

After the WireMock server is started, it's time to create a stub like the following.

```groovy
stubFor(get(urlEqualTo(gitHubAccessTokenUri + gitHubAccessTokenRequest.toQueryString()))
        .willReturn(aResponse()
        .withBody("access_token=${accessToken}&expires_in=28800&refresh_token=${refreshToken}&token_type=Bearer")))
```

The above stub is for simulating a GitHub access token service endpoint. So, in this scenario, GitHub is an external 
identity provider service. As long as there is not a path conflict, WireMock will respond to the ```gitHubAccessTokenUri```
with the content specified in the stub definition above.

The only outstanding question is then, "How does ```gitHubAccessTokenUri``` get populated?" In our tests, WireMock
config is specified (see [Spring's @Value injection](https://www.baeldung.com/spring-value-annotation)) in an ```application-test.yml``` file located in the ```test/resources``` directory. And to ensure that the
_test_ profile is active during automated tests, we have the following class in a package under the ```test/groovy``` 
directory.

```groovy
class TestProfileResolver implements ActiveProfilesResolver {

  @Override
  String[] resolve(Class<?> testClass) {
    ['test']
  }
}
```

### Using Mock Services With Spring Boot ###

Orignally, we planned on using a mock server for local development and for tests. But setting up WireMock for local development
comes with some additional baggage (it's not as trivial to setup as it is with tests). Certainly, this impediment could be 
overcome. But it also forced the question, "Is a mock server the best solution for local development?" It turned out, for us,
the answer was "no."

Test suites have very specific service mocking requirements that may vary from test suite to test suite. WireMocks was
perfect for this use case; each test suite could define its own stubs. However, for a local development environment, different
concerns must be addressed; consistency and availability of the mock services is the primary concern. For that reason,
we chose to use Spring controllers as mock services and change the URL configs to point to those controllers using a _dev_
profile. We just had to make sure that there were no URI path collisions. Oh, and we had to make sure the mock services
were only available when the _dev_ profile was active.

Here's a snippet of the application-dev.yml configuration file.

```yaml
auth:
  provider:
    github:
      access_token_uri: "http://localhost:8080/mock/github/login/oauth/access_token"
```

And here's the associated Spring controller code.

```java
@RestController
@Profile("dev") //only active with the dev profile
public class GitHubStubController {

  @GetMapping("${auth.provider.github.access_token_uri}")
  String accessToken() {
    return "access_token=TH3M0NK3Y3@T5CU5T@RD&expires_in=28800&refresh_token=TH3E@GL3FL135@TD@WN&token_type=Bearer";
  }
  
  ...
}
```

### Summary ###

* Service endpoint mocks can be invaluable for both automated tests and for local development
* WireMock is a nice mock server that can be easily integrated into automated test suites
* For local development, consider using profile dependent service mocks as a simple service endpoint mocking solution
