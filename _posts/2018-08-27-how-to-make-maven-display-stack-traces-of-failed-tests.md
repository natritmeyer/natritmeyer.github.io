---
layout: single
title: How to make maven display stack traces of failed tests
date: 2018-08-21
categories: howto
tags: maven surefire failsafe
excerpt: "The use of and history behind surefire's trimStackTrace feature"
---

## TL;DR

If you want to see stack traces when tests executed by `mvn test ` or `mvn verify` fail instead of the short summary you're seeing, make sure you've got `trimStackTrace` set to `false` in your `pom.xml`.

E.g.

### Enable stack traces for `mvn test`

```xml
<build>
  <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${your.maven.surefire.version.here}</version>
        <configuration>
            <trimStackTrace>false</trimStackTrace>
        </configuration>
    </plugin>
  </plugins>
</build>
```

### Enable stack traces for `mvn verify`

```xml
<build>
  <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-failsafe-plugin</artifactId>
        <version>${your.maven.failsafe.version.here}</version>
        <configuration>
            <trimStackTrace>false</trimStackTrace>
        </configuration>
        <executions>
          <!-- usual execution stuff here -->
        </executions>
    </plugin>
  </plugins>
</build>
```

And now some archaeology...

## The history behind `trimStackTrace`

Back in the day `mvn test` and `mvn verify` would print the stack trace of any test that failed. Though the information in the stack trace was useful for debugging, stack traces took up rather a lot of space in the console output. Particularly if there were a number of failing tests it became easy to get lost in the noise. 

In a wonderfully terse description of the problem, [Issue 936](https://issues.apache.org/jira/browse/SUREFIRE-936) in surefire's Jira project explains,

>It is too hard to find out what failed at a glance. Fix this.

A [bit of coding](https://github.com/apache/maven-surefire/commit/fff9e32febbc21602b6a393ce448d6caddf62fb5) later and surefire version 2.13 was released back in [December 2012](https://mvnrepository.com/artifact/org.apache.maven.plugins/maven-surefire-plugin/2.13). Amongst the 'notable changes' listed in the [announcement](https://lists.apache.org/thread.html/e0d7378b1b272aab59662f88f555e2d4aa5c705098b2b62f31a296da@1356657888@%3Cdev.maven.apache.org%3E) was,

> Stack trace trimming now works for JUnit4.x. The itch you didn't know you had until it was scratched.

The previously-noisy stack trace output was replaced with a calmer [1-line error summary](https://maven.apache.org/surefire/maven-surefire-plugin/newerrorsummary.html). And just like that everyone waved goodbye to their stack traces.

Speaking personally, it's nice to have the option. I'd have preferred the old behaviour remained the default, but it doesn't matter much either way.
