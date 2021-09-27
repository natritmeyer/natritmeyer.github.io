---
layout: single
title: Cucumber-JVM Dependency Injection with Spring
date: 2019-01-13
categories: howto
tags: [maven, cucumber, dependency injection, spring]
excerpt: "How to set up your Cucumber JVM project to use Dependency Injection with Spring"
---

## Introduction

One of the common challenges with using cucumber is passing around the `WebDriver` instance, as well as page object instances. I've seen all sorts of weird ways people have dealt with the problem, but all these solutions have one thing in common: they get tied in knots. The pattern established early in the project doesn't work with a new scenario, so a hack is put in place to keep things working. After a few more similar incidents the framework falls in a heap. Testing gets out of sync with dev, falls further and further behind, and it's not long before test automation is abandoned. This is not a rare occurrence; I've seen it time and time again.

The solution to this problem is [Dependency Injection](https://martinfowler.com/articles/injection.html).

What does that mean? Well, instead of creating and managing instances of `WebDriver` yourself, use a library to do all that for you, including making sure a `WebDriver` instance ends up where you need it. Same goes for Page Objects. Instead of creating an instance of a page object, somehow getting your `driver` into it and then getting the page object instance where it needs to be, let the same library do that for you too.

The library we're going to do this with is [Spring](https://spring.io/projects/spring-framework) and the pattern we're going to use is simple.

In this post we'll go through what's required to put together a test automation framework with Cucumber-JVM that utilises dependency injection using Spring. We'll go through each step, starting from nothing, explaining what the various moving parts are. As such this tutorial is aimed more at beginners than those with more experience. There's going to be some fudging of the detail when it comes to some of spring's more involved concepts - those with more experience will be able to point out the hand waving. YMMV.

Also, the finished product lives here: [https://github.com/natritmeyer/cucumber-jvm-dependency-injection-with-spring](https://github.com/natritmeyer/cucumber-jvm-dependency-injection-with-spring).

## Framework overview

To keep things focused we're going to keep the "feature" tested in our framework very simple and uninteresting: It'll search [Bing](https://www.bing.com) for the word "cucumber", and then verify that the link to the first hit includes the word "Cucumber". Here's the contents of the feature file:

```gherkin
Feature: Search for content

  Scenario: Search for information
    Given I am on the bing search engine
    When I enter a search term
    Then relevant results for that search term are displayed
```

Our aim is that our single scenario will be able to test both desktop *and* mobile implementation of Bing's search function, and that we'll use dependency injection to make sure the right WebDriver instance and implementation-specific page objects are used in the step definition class.

### Ingredients

The framework we're going to build is based on the following:

* Java
* Maven
* Cucumber-JVM
* Spring
* WebDriver
* Chrome

I'm assuming you've got at least a vague notion of what those are.

OK, let's start putting the pieces together.

## Step by step

We'll start by creating an ordinary, bare bones cucumber project that'll be just enough to execute the feature file. Once that's in place we'll add what we need to integrate spring, and finally implement the springified page objects and browser management classes.

We'll begin by creating the root directory of our project; let's call it `cucumber-jvm-dependency-injection-with-spring`

### Setting up a basic cucumber project

Maven projects have a `pom.xml` file at their root. Amongst other things the `pom.xml` file describes the project, lists the project's dependencies, and lists and configures any maven plugins that assist in building the software. Use your IDE to create an empty Maven project, and open the autogenerated `pom.xml` file in the root (feel free to copy and paste from the one in the github repo linked to at the end of this post).

#### The `pom.xml` file

Let's start by adding the usual dependencies required of all cucumber projects:

```xml
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-junit</artifactId>
  <version>4.2.1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-java</artifactId>
  <version>4.2.1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-java8</artifactId>
  <version>4.2.1</version>
  <scope>test</scope>
</dependency>
```

Next we need to configure where in the maven lifecycle the tests will execute - for this example project we'll have them run during the `verify` phase. To ensure that, as well as telling the compiler that we're writing Java 8 code, we'll add the following to the pom, just below the end of the `dependencies` section:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.8.0</version>
      <configuration>
        <source>1.8</source>
        <target>1.8</target>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <version>2.22.1</version>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

With that done we can move on to the next stage of creating a basic cucumber project.

#### The feature file

The feature file should be called `search.feature` and placed in `src/test/resources/features/`. Once created set its content to be:

```gherkin
Feature: Search for content

  Scenario: Search for information
    Given I am on the bing search engine
    When I enter a search term
    Then relevant results for that search term are displayed
```

#### The JUnit test runner

We now need a file called `RunCucumberIT.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/testrunners/`; here's what its content needs to be:

```java
package com.natritmeyer.examplecucumberjvmspringframework.testrunners;

import cucumber.api.CucumberOptions;
import cucumber.api.junit.Cucumber;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
  features = {"classpath:features/"},
  plugin   = {"pretty"},
  glue     = {"com.natritmeyer.examplecucumberjvmspringframework.steps"})
public class RunCucumberIT {
}
```

That's just telling cucumber to look for the feature files in `src/test/resources/features/`, and to look for the step definitions in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/steps/` (you're right, we haven't made the second directory yet).

#### The step definitions class

The last file we need to complete our basic cucumber project is the step definition class. It should be called `BingSearchSteps.java` and put here: `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/steps/`. Here's what it needs to look like:

```java
package com.natritmeyer.examplecucumberjvmspringframework.steps;

import cucumber.api.java8.En;

public class BingSearchSteps implements En {
  public BingSearchSteps() {
    Given("I am on the bing search engine", () -> {
    });

    When("I enter a search term", () -> {
    });

    Then("relevant results for that search term are displayed", () -> {
    });
  }
}
```

And with that we have something that will run. At the command line you should be able to run `mvn verify` and get some meaningful output.

We're now going to get to the interesting part - adding spring into the mix, followed by webdriver, browser configuration, page objects, and clean step definition code.

### Adding Spring to the project

To use dependency injection we need to add a few things to the project. Let's start with a couple of dependencies.

#### Two more dependencies for the `pom.xml` file

Add the following dependencies to the `pom.xml` file:

```xml
<dependency>
  <groupId>io.cucumber</groupId>
  <artifactId>cucumber-spring</artifactId>
  <version>4.2.1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>5.1.4.RELEASE</version>
  <scope>test</scope>
</dependency>
```

The first of those is a cucumber library that allows the use of spring in cucumber projects; the second is the tiny portion of the massive Spring ecosystem that cucumber needs to allow dependency injection.

Now we need to configure cucumber spring.

#### Adding `cucumber.xml`

We now need to make a file called `cucumber.xml` and place it in `src/test/resources/`. The purpose of this file is to tell cucumber's spring interaction where to look for things. Here's the contents of the file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context="http://www.springframework.org/schema/context"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
  http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">

  <context:annotation-config/>
  <bean name="Config" class="com.natritmeyer.examplecucumberjvmspringframework.config.Config"/>
</beans>
```

The last two lines are the important ones. The first of them tells spring to use "annotation config", which basically means we can manage all our spring configuration in code rather than in this xml file. The second line tells spring to look for a class called `Config` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/config/`. And it's *that* class which really controls what's happening in spring, so let's move on to that.

#### The spring configuration class

Let's create a file called `Config.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/config/` and give it the following content:

```java
package com.natritmeyer.examplecucumberjvmspringframework.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.PropertySource;

@PropertySource("application.properties")
@ComponentScan("com.natritmeyer.examplecucumberjvmspringframework")
public class Config {
}
```

Just to explain what's going on here, the first line (`@PropertySource`) tells spring to look for configuration values in a file called `application.properties`, and the second line tells spring to look for things it's interested in in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/`. The sorts of things spring is interested in that it'll find in this project are classes annotated with `@Component` and `@Configuration`. We'll get more into the detail of that when we start creating page objects and browsers a bit later.

And that's what's required to get spring added to the project. From now on we're going to be spending our time writing step definitions that have their dependencies injected in, and writing page objects that do the same.

### Filling out the test

We'll start this phase by adding the last of our dependencies.

#### The remaining `pom.xml` dependencies

We're going to use AssertJ for our assertions, and WebDriver to control the browser. Here's what needs adding to the `pom.xml` file:

```xml
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <version>3.11.1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.seleniumhq.selenium</groupId>
  <artifactId>selenium-java</artifactId>
  <version>3.141.59</version>
  <scope>test</scope>
</dependency>
```

#### Creating the AUT model

So that the step definitions will work against Bing's mobile and desktop search engine implementations it's important that we "code to an interface, not an implementation". This means that the steps must only deal with _interfaces_, not _classes_. As such we need to model the application under test (i.e. Bing) as a group of interfaces.

If we remember our scenario, all that's required is the ability to interact with Bing's home page and its search results page. Specifically, we need to load the Bing home page and use it to search for a search term, and we need to get the title of the first link on the search results page.

We need to create a file to contain the first interface called `BingHomePage.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/aut/model/`. Its contents are:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.model;

public interface BingHomePage {
  void load();

  void searchFor(String cucumber);
}
```

The other interface needs to go in a file named `BingSearchResultsPage.java` in the same directory as the interface above. The file's contents should be:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.model;

public interface BingSearchResultsPage {
  String getFirstResultTitle();
}
```

With that we have our complete model for our application under test.

#### Injecting the model into the step definitions

We're now at the stage where we can update our step definition so that its dependencies are injected. Here's how that works...

First, the constructor is updated. Here's how it looked originally:

```java
public BingSearchSteps() {
  //...
}
```

We need to change it so it looks like this:

```java
@Autowired
public BingSearchSteps(BingHomePage bingHomePage, BingSearchResultsPage bingSearchResultsPage) {
  //...
}
```

There are a couple of changes. Firstly, we've annotated the constructor with `@Autowired`. This tells spring that when an instance of this step definitions class is created, automatically pass in (or 'inject') pre-created instances of whatever's mentioned as constructor arguments. Secondly, we've added one instance of each of our AUT model interfaces as constructor arguments. In reality, this means that when cucumber is executed, spring will automagically create instances of the `BingHomePage` and `BingSearchResultsPage` interfaces (how it does this we'll get to later), stores them in a pool of managed objects (so they can be used across the spring context), and then pass those instances into the step definition class' constructor.

We can now update the steps themselves from:

```java
Given("I am on the bing search engine", () -> {
});

When("I enter a search term", () -> {
});

Then("relevant results for that search term are displayed", () -> {
});
```

...to...

```java
Given("I am on the bing search engine", () -> {
  bingHomePage.load();
});

When("I enter a search term", () -> {
  bingHomePage.searchFor("cucumber");
});

Then("relevant results for that search term are displayed", () -> {
  assertThat(bingSearchResultsPage.getFirstResultTitle()).contains("Cucumber");
});
```

You'll notice that the steps here are _very_ simple. They're just one line each calling only what's available on the interfaces that represent the application under test. As a result they're readable - a key feature of good step definition files.

Here's the full step definition class after being springified:

```java
package com.natritmeyer.examplecucumberjvmspringframework.steps;

import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingHomePage;
import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingSearchResultsPage;
import cucumber.api.java8.En;
import org.springframework.beans.factory.annotation.Autowired;

import static org.assertj.core.api.Assertions.assertThat;

public class BingSearchSteps implements En {
  private final BingHomePage bingHomePage;
  private final BingSearchResultsPage bingSearchResultsPage;

  @Autowired
  public BingSearchSteps(BingHomePage bingHomePage, BingSearchResultsPage bingSearchResultsPage) {
    this.bingHomePage = bingHomePage;
    this.bingSearchResultsPage = bingSearchResultsPage;

    Given("I am on the bing search engine", () -> {
      bingHomePage.load();
    });

    When("I enter a search term", () -> {
      bingHomePage.searchFor("cucumber");
    });

    Then("relevant results for that search term are displayed", () -> {
      assertThat(bingSearchResultsPage.getFirstResultTitle()).contains("Cucumber");
    });
  }
}
```

#### Creating implementations of the AUT model

Before the steps can be used we need at least one implementation of the AUT model we've created. In this example cucumber project we'll create two separate implementations of the AUT model interfaces - desktop and mobile. Each of our two interfaces represents a page, so each implementation of an interface represents a page object. So, we're going to end up with 4 new classes: `DesktopBingHomePage`, `DesktopBingSearchResultsPage`, `MobileBingHomePage`, and `MobileBingSearchResultsPage`.

#### Desktop page objects

We'll begin with the desktop implementation of the AUT interfaces, starting with the home page.

We'll start be creating a file called `DesktopBingHomePage.java` in `src/test/java/com/natritmeyer/examplecucumnberjvmspringframework/`. Here are the file's contents:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.implementations.desktop;

import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingHomePage;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("desktop")
public class DesktopBingHomePage implements BingHomePage {
  @Value("${aut.urls.home}")
  private String homePageUrl;

  private final WebDriver driver;

  @Autowired
  public DesktopBingHomePage(WebDriver driver) {
    this.driver = driver;
  }

  @Override
  public void load() {
    driver.get(homePageUrl);
  }

  @Override
  public void searchFor(String searchTerm) {
    driver.findElement(By.id("sb_form_q")).sendKeys(searchTerm);
    driver.findElement(By.id("sb_form_go")).click();
  }
}
```

There are a few things here that need explaining.

`@Component` tells spring to create an instance of this class and add it to the pool of objects spring manages. Wherever `DesktopBingHomePage` is mentioned as a constructor argument (and the constructor is annotated with `@Autowired`) this instance will be injected in.

The `@Profile("desktop")` annotation acts as a kind of filter - this is how spring chooses between multiple implementations of an interface. If there was another class that implemented `BingHomePage` (and there will be in a minute when we create a mobile version), spring will choose between them based on the `spring.profiles.active` system property. We'll see how to go about setting that a bit later.

This page object that represents Bing's home page for desktops `implements BingHomePage`. I.e. it's an implementation of the `BingHomePage` interface. This allows `DesktopBingHomePage` to be injected into any constructor that asks for `BingHomePage`.

The page object has a constructor that's annotated with `@Autowired`. So, when spring automagically creates an instance of this class (because it's annotated with the `@Component` annotation - see above), it'll inject it with a preexisting WebDriver instance. We'll get to WebDriver management in a bit.

There are two methods in this class, both annotated `@Override`: `public void load()` and `public void searchFor(String searchTerm)`. These are the methods that fulfil the requirements of the `BingHomePage` interface - they're what actually get used by the step definition when the test is run.

Finally, there's the `homePageUrl` variable with its `@Value("${aut.urls.home}")` annotation. This is how to get config data into your object. Here, the `homePageUrl` value is being populated with the contents of whatever `aut.urls.home` is set to in the `src/test/resources/application.properties` file. Let's take a quick diversion into configuration...

#### A diversion through the `application.properties` configuration file

The `@Value(...)` annotation is used to get data from configuration files.

Back when we were creating the `Config` class we annotated it with `@PropertySource("application.properties")`. This instructed spring to look for any config in `src/test/resources/application.properties` during the course of test execution. So, create that file in that location, and set its contents to the following:

```properties
aut.urls.home=https://www.bing.com/
```

Now, when the tests get run, the `DesktopBingHomePage`'s `homePageUrl` string will have its value set to `https://www.bing.com/`. We'll make more use of the config file when we get to configuring our desktop and mobile browsers.

#### The rest of the page objects

The other 3 page objects are just variations on the same theme, so here they are:

Create a file called `DesktopBingSearchResultsPage.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/aut/implementations/desktop/`, and set its contents to:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.implementations.desktop;

import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingSearchResultsPage;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("desktop")
public class DesktopBingSearchResultsPage implements BingSearchResultsPage {
  private final WebDriver driver;

  @Autowired
  public DesktopBingSearchResultsPage(WebDriver driver) {
    this.driver = driver;
  }

  @Override
  public String getFirstResultTitle() {
    return driver.findElement(By.cssSelector("ol#b_results > li > h2 > a")).getText();
  }
}
```

Create a file called `MobileBingHomePage.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/aut/implementations/mobile/`, and set its contents to:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.implementations.mobile;

import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingHomePage;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("mobile")
public class MobileBingHomePage implements BingHomePage {
  @Value("${aut.urls.home}")
  private String homePageUrl;

  private final WebDriver driver;

  @Autowired
  public MobileBingHomePage(WebDriver driver) {
    this.driver = driver;
  }

  @Override
  public void load() {
    driver.get(homePageUrl);
  }

  @Override
  public void searchFor(String searchTerm) {
    driver.findElement(By.id("hc_popnow")); //wait for the trending stories - js has done loading by this point
    driver.findElement(By.id("sb_form_q")).sendKeys(searchTerm);
    driver.findElement(By.id("sbBtn")).click();
  }
}
```

Create a file called `MobileBingSearchResultsPage.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/aut/implementations/mobile/`, and set its contents to:

```java
package com.natritmeyer.examplecucumberjvmspringframework.aut.implementations.mobile;

import com.natritmeyer.examplecucumberjvmspringframework.aut.model.BingSearchResultsPage;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("mobile")
public class MobileBingSearchResultsPage implements BingSearchResultsPage {
  private final WebDriver driver;

  @Autowired
  public MobileBingSearchResultsPage(WebDriver driver) {
    this.driver = driver;
  }

  @Override
  public String getFirstResultTitle() {
    driver.findElement(By.cssSelector("ol#b_results")); //waits for the results to be displayed
    return driver.findElement(By.cssSelector("ol#b_results > li.b_algo > div > a > h2")).getText();
  }
}
```

Note that the last two are the mobile implementations of the `BingHomePage` and `BingSearchResultsPage` interfaces are are annotated with `@Profile("mobile")` instead of `@Profile("desktop")`.

### Browser management

You'll have noticed that the page objects all have an `@Autowired` constructor that takes an instance of `WebDriver`. We'll now code up what's behind that.

When we run the tests against Bing's mobile search page we need a webdriver session set up with a mobile user agent, and with window dimensions of a mobile browser. Conversely, when we run the tests against Bing's desktop site we'll want webdriver set up with desktop dimensions. To achieve this we'll create two `@Bean` methods - these are methods that return instances of specific classes that can be called by spring when instances of specific classes are required. As before we'll start with the desktop and move on to the mobile implementation.

#### Desktop browser

We'll begin by creating a file called `DesktopBrowser.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/browsers/` and we'll set its content to this:

```java
package com.natritmeyer.examplecucumberjvmspringframework.browsers;

import org.openqa.selenium.Dimension;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.TimeUnit;

@Configuration
public class DesktopBrowser {
  @Value("${browser.desktop.width}")
  private int desktopWidth;

  @Value("${browser.desktop.height}")
  private int desktopHeight;

  @Bean
  @Profile("desktop")
  public WebDriver getDriver() {
    WebDriver driver = new ChromeDriver();
    driver.manage().window().setSize(new Dimension(desktopWidth, desktopHeight));
    driver.manage().timeouts().implicitlyWait(10, TimeUnit.SECONDS);
    return driver;
  }
}
```

There are a few things going here - let's go through them.

The class is annotated with `@Configuration`. This tells spring to look in this class for a method annotated with `@Bean`.

There are two `int` variables, `desktopWidth` and `desktopHeight`. The value of these ints will come from `application.properties`; we may as well add the relevant values now. Here's what needs to go into the properties file:

```properties
browser.desktop.width=1024
browser.desktop.height=768
```

Back in the `DesktopBrowser` class, there's a method called `public WebDriver getDriver()`. Because this is annotated with `@Bean`, spring will use this method to get its `WebDriver` instance, and then use that whenever any constructor asks for a `WebDriver` instance - this is how using dependency injection gets rid of the problem of passing `driver` around in weird ways leading to project collapse.

The final thing to note about the class is that the `getDriver()` method is also annotated with `@Profile("desktop")`, just like the first two page object classes we created. This is how spring avoids confusing this `@Bean` method that returns a `WebDriver` instance, and the next one we're going to create.

#### Mobile browser

When we're running the tests against Bing's mobile site we want to make sure we're using a webdriver instance that's configured to look and act like a mobile browser. For the purposes of this example project it's enough to just set the window's width and height, along with setting the user agent to something mobile.

So, let's create our final file, `MobileBrowser.java` in `src/test/java/com/natritmeyer/examplecucumberjvmspringframework/browsers/` and give it the following contents:

```java
package com.natritmeyer.examplecucumberjvmspringframework.browsers;

import org.openqa.selenium.Dimension;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import java.util.concurrent.TimeUnit;

@Configuration
public class MobileBrowser {
  @Value("${browser.mobile.width}")
  private int mobileWidth;

  @Value("${browser.mobile.height}")
  private int mobileHeight;

  @Value("${browser.mobile.useragent}")
  private String userAgent;

  @Bean
  @Profile("mobile")
  public WebDriver getDriver() {
    String userAgentString = String.format("--user-agent=%s", userAgent);

    ChromeOptions chromeOptions = new ChromeOptions();
    chromeOptions.addArguments(userAgentString);

    WebDriver driver = new ChromeDriver(chromeOptions);
    driver.manage().window().setSize(new Dimension(mobileWidth, mobileHeight));
    driver.manage().timeouts().implicitlyWait(10, TimeUnit.SECONDS);
    return driver;
  }
}
```

Just like the `DesktopBrowser` class, this class needs some config adding to `application.properties`. Here's what you need to add:

```properties
browser.mobile.width=400
browser.mobile.height=800
browser.mobile.useragent=Mozilla/5.0 (iPhone; CPU OS 11_0 like Mac OS X) AppleWebKit/604.1.25 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1
```

The contents of `MobileBrowser`'s `getDriver()` method is different to `DesktopBrowser`'s, but it's just because it has to handle the user agent string as well as the window dimensions - nothing more interesting than that.

And... that's it. That's everything. We're at the stage now where everything should work...

## Executing the feature file

With our feature file, step definitions, page objects, and browser management in place, we're ready to execute the feature file.

Let's first make sure we're in the right place:

```sh
cd cucumber-jvm-dependency-injection-with-spring
```

We're now ready to run. Let's start with the desktop implementation. To execute the feature file against Bing's desktop site we need to set the `desktop` profile. The command that executes the feature files and sets that profile is as follows:

```sh
mvn clean verify -Dspring.profiles.active=desktop
```

If you run that you should see a browser flash up on the screen, navigate to [bing.com](https://www.bing.com), search for "cucumber", and then display the search results before closing.

As well as that you should see something like this in your console:

```console
...
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.natritmeyer.examplecucumberjvmspringframework.testrunners.RunCucumberIT
Starting ChromeDriver 2.45.615355 (d5698f682d8b2742017df6c81e0bd8e6a3063189) on port 19432
Only local connections are allowed.
Jan 15, 2019 11:59:45 AM org.openqa.selenium.remote.ProtocolHandshake createSession
INFO: Detected dialect: OSS
Feature: Search for content

  Scenario: Search for information                           # features/search.feature:3
    Given I am on the bing search engine                     # BingSearchSteps.java:19
    When I enter a search term                               # BingSearchSteps.java:23
    Then relevant results for that search term are displayed # BingSearchSteps.java:27

1 Scenarios (1 passed)
3 Steps (3 passed)
0m3.515s

[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.653 s - in com.natritmeyer.examplecucumberjvmspringframework.testrunners.RunCucumberIT
...
```

If that's working then go ahead and execute the feature file again, but this time with the `mobile` profile set:

```sh
mvn clean verify -Dspring.profiles.active=mobile
```

If that worked you'll have seen a much smaller browser window open, navigate to the Bing home page, searched using the mobile interface, and displayed the mobile search results page before closing. The console output should be practically identical.

## Conclusion

That's it. We created an example cucumber project with a single feature file backed by a single step definition class. Because we set up that step definition class to use interfaces rather than classes we were able to inject in whatever implementation we liked - in this case `desktop` or `mobile` page object implementations of those interfaces. By injecting appropriately configured WebDriver instances into those implementation-specific page objects we were able with a single command line switch (`spring.profiles.active`) to determine at runtime which of the implementations we wanted to test.

Most importantly, by using dependency injection we got rid of the need to create and maintain WebDriver and page object instances, removing the need to carefully pass them around.

This is a massive maintenance win, as many readers will appreciate. It also increased the readability of the code (especially the step definition class), as well as divided up the project into loosely coupled code.

Note, this example project is not perfect - far from it. There are quire a few optimisations I'd ordinarily make, but for the sake of clarity I've left them out. However, it should be good enough to show you how cucumber and spring can work together to achieve dependency injection in most scenarios.

Finally, this example project in its complete form can be found here [https://github.com/natritmeyer/cucumber-jvm-dependency-injection-with-spring](https://github.com/natritmeyer/cucumber-jvm-dependency-injection-with-spring).