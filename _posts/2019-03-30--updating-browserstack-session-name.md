---
layout: single
title: Updating a BrowserStack session's name
date: 2019-03-30
categories: howto
tags: [maven, spring-flux, webclient, browserstack]
excerpt: "How to update a BrowserStack's session name using Spring Flux's WebClient"
---

## Intro

BrowserStack's [REST API](https://www.browserstack.com/automate/rest-api) provides the ability to update various aspects of a session, e.g. test result, test name, etc. They do seem to provide a client library called `browserstack-automate-java` for interacting with the API ([GitHub](https://github.com/browserstack/browserstack-automate-java), [maven-central](https://mvnrepository.com/artifact/com.browserstack/automate-client-java/0.4)), but it doesn't seem to get much love and I can't find any documentation for it. Its basic functions appear to work but using it to update anything in browserstack seems to fail.

## Solving the problem

Wanting to update the session name and result, I wrote my own client based on Spring-Flux's [WebClient](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client). Here's what you need if you want to do the same.

### Some dependencies

First, you'll need some dependencies:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webflux</artifactId>
    <version>5.1.5.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.4</version>
  </dependency>
  <dependency>
    <groupId>io.projectreactor.netty</groupId>
    <artifactId>reactor-netty</artifactId>
    <version>0.8.5.RELEASE</version>
  </dependency>
</dependencies>
```

### Getting the BrowserStack Session ID

To update a BrowserStack session you need its ID; you can get that from your `RemoteWebDriver` instance:

```java
RemoteWebDriver driver = ...;

// do some navigation, testing, etc.

String sessionId = driver.getSessionId().toString();

// sessionId is now "d430d61935a66ddd9c5c9a9e15362c520414b082"
```

### Preparing the payload

Next, because you're going to be sending JSON you'll need a simple class that when converted to JSON matches what the API expects. Given we're updating the session name and status the following is enough for our needs:

```java
public class SessionDetails {
  private String name;
  private String status;

  private SessionDetails(String name, String status) {
    this.name = name;
    this.status = status;
  }

  public String getName() {
    return name;
  }

  public String getStatus() {
    return status;
  }
}
```

Assuming you're using cucumber-jvm you'll probably be making calls to BrowserStack's API in an `@After` step. You'll therefore have access to the scenario's name as well as its result and can set the BrowserStack session's result accordingly:

```java
@After
public void setBrowserStackSessionDetails(Scenario scenario) {
  String testName = scenario.getName();
  
  // The API expects either "passed" or "failed". Here's a
  // rubbish way to ensure that. Maybe use an enum instead.
  String testResult = scenario.isFailed() ? "failed" : "passed";
}
```

We can now create an instance of the `SessionDetails` with the details we want to send:

```java
@After
public void setBrowserStackSessionDetails(Scenario scenario) {
  String testName = scenario.getName();
  String testResult = scenario.isFailed() ? "failed" : "passed";
  
  SessionDetails sessionDetails = new sessionDetails(testName, testResult);
}
```

## Making the call to BrowserStack

Now that we've got something we want to send to BrowserStack we need to get a HTTP client ready.

## Preparing the WebClient

The old `resttemplate` way of doing things will be [deprecated in the near future](https://docs.spring.io/spring/docs/5.1.5.RELEASE/javadoc-api/org/springframework/web/client/RestTemplate.html) and replaced with `WebClient`. Its API isn't all that intuitive but it makes sense in the end. Here's how to set up a `WebClient``:

```java
import org.springframework.web.reactive.function.client.ExchangeFilterFunctions;
import org.springframework.web.reactive.function.client.WebClient;

//...

String browserStackUsername = "MY_BROWSERSTACK_USERNAME";
String browserStackAccessKey = "MY_BROWSERSTACK_ACCESS_KEY";

WebClient client = WebClient.builder()
  .baseUrl("https://api.browserstack.com")
  .filter(ExchangeFilterFunctions.basicAuthentication(browserStackUsername, browserStackAccessKey))
  .build();
```

### Updating BrowserStack's session details

The final thing to do is make the call to browserstack. Here's how it's done:

```java
import org.springframework.http.MediaType;
import reactor.core.publisher.Mono;

//...

client.put().uri(builder -> builder.path("/automate/sessions/" + sessionId + ".json").build())
  .contentType(MediaType.APPLICATION_JSON)
  .body(Mono.just(sessionDetails), SessionDetails.class)
  .retrieve()
  .bodyToMono(String.class)
  .block();
```

That takes the `sessionDetails` containing the test name and status (passed/failed), converts it to JSON, and makes a PUT call to the session ID extracted from the `RemoteWebDriver` earlier.

Given your scenario name was "My Scenario Name" and the scenario failed, here's what you should see:

Here's what you should see:

![Updated BrowserStack session details](/assets/images/updated_browserstack_session.png)

Hope that helps.