---
layout: single
title: Reporting aggregated unit and integration test coverage with jacoco
date: 2019-06-26
categories: howto
tags: [maven, jacoco, surefire, failsafe, coverage]
excerpt: "How to configure Jacoco to create coverage reports and checks based on aggregated unit and integration test coverage data"
---

>The higher your test coverage, the less your fear.
>
<cite>Robert C. Martin, Clean Code (2009), 124.</cite>

Jacoco is a pretty popular code coverage tool in the java world. Configuring
it in the `pom.xml` to generate coverage reports for unit and integration is
easy enough. But, there are times when you want a report that takes both unit
_and_ integration tests into account.

In this post we'll go through the configuration options required to generate
unit and integration test coverage reports as well as a report that integrates
the merged coverage data. I'm assuming the following setup:

* Java 8+
* Maven 3.3+

# Unit test coverage report

Let's start by configuring jacoco to create a unit test coverage report. The
first step is adding jacoco to your project.

## Include the Jacoco plugin

To add jacoco to your project add the following to the `<build>` section of
your `pom.xml`:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.4</version>
      <executions>
      </executions>
    </plugin>
  </plugins>
</build>
```

With that it's time to add a load of `<execution>` blocks to the `<executions>`
section.

## Configuring jacoco unit test coverage data

Jacoco splits collection of test coverage data from the generation of the
coverage report. There are therefore `<execution>` blocks required for each
stage in the process.

This first execution block tells jacoco where to store the unit test coverage
data it will collect during unit test execution:

```xml
<execution>
  <id>before-unit-test-execution</id>
  <goals>
    <goal>prepare-agent</goal>
  </goals>
  <configuration>
    <destFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</destFile>
    <propertyName>surefire.jacoco.args</propertyName>
  </configuration>
</execution>
```

The `<destFile>` element contains the path the data will be written to. The
`<propertyName>` contains the name of a variable that will be populated with
some arguments which will be passed to surefire (explained later) to
point it at the coverage data collection file defined in `<destFile>`.

## Generating the unit test coverage report

The next execution block is used to tell jacoco where to write the unit test
report, and which coverage data file to generate the report from.

```xml
<execution>
  <id>after-unit-test-execution</id>
  <phase>test</phase>
  <goals>
    <goal>report</goal>
  </goals>
  <configuration>
    <dataFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</dataFile>
    <outputDirectory>${project.reporting.outputDirectory}/jacoco-unit-test-coverage-report</outputDirectory>
  </configuration>
</execution>
```

The report generator will look for its input (the coverage data file populated
during unit testing) in the path defined in the `<dataFile>` element, and will
save the generated report in the path specified in `<outputDirectory>`. Note
that in the above example uses the `project.reporting.outputDirectory` property
which points at `target/site`.

## Configuring surefire

Maven uses surefire to execute unit tests, and for coverage data to be collected
jacoco configures surefire to use a java agent that instruments the classes
under test to enable collection of execution data.

Here's the how surefire needs to be configured:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>2.22.2</version>
  <configuration>
    <argLine>${surefire.jacoco.args}</argLine>
  </configuration>
</plugin>
```

Now, next time you run `mvn test` you should see the following:

```txt
[INFO] --- jacoco-maven-plugin:0.8.4:prepare-agent (before-unit-test-execution) @ jacocoexample ---
[INFO] surefire.jacoco.args set to -javaagent:/Users/nat/.m2/repository/org/jacoco/org.jacoco.agent/0.8.4/org.jacoco.agent-0.8.4-runtime.jar=destfile=/Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-unit-tests.exec
[INFO] 
...
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ jacocoexample ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.natritmeyer.jacocoexample.unit.ThingTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.047 s - in com.natritmeyer.jacocoexample.unit.ThingTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
[INFO] --- jacoco-maven-plugin:0.8.4:report (after-unit-test-execution) @ jacocoexample ---
[INFO] Loading execution data file /Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-unit-tests.exec
[INFO] Analyzed bundle 'jacocoexample' with 3 classes
```

Hopefully you can make out that the `surefire.jacoco.args` property is set in
the first step. Next comes the test execution, followed by the generation of
the unit test coverage report. If you look in your `target/site/jacoco-unit-test-coverage-report/`
directory you should see an `index.html` file. Open it and you should see your
unit test coverage report:

![Jacoco Unit Test Coverage Report](/assets/images/jacoco_unit_test_coverage_report.png)

# Integration test coverage report

Now that we've got our build generating a coverage report for the unit tests
it's time to set up matching config for the integration tests.

In principle this is exactly the same as setting up coverage for unit tests with
only one difference: instead of configuring `surefire`, the unit tests execution
tool, we're going to configure `failsafe`, maven's integration test execution
tool.

## Configuring jacoco's integration coverage report

Here are the two `<execution>` blocks required to set up integration test coverage:

```xml
<execution>
  <id>before-integration-test-execution</id>
  <phase>pre-integration-test</phase>
  <goals>
    <goal>prepare-agent</goal>
  </goals>
  <configuration>
    <destFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</destFile>
    <propertyName>failsafe.jacoco.args</propertyName>
  </configuration>
</execution>

<execution>
  <id>after-integration-test-execution</id>
  <phase>post-integration-test</phase>
  <goals>
    <goal>report</goal>
  </goals>
  <configuration>
    <dataFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</dataFile>
  <outputDirectory>${project.reporting.outputDirectory}/jacoco-integration-test-coverage-report</outputDirectory>
  </configuration>
</execution>
```

## Configuring failsafe

Finally, `failsafe` needs to be configured to use the contents of the
`failsafe.jacoco.args` property as arguments:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-failsafe-plugin</artifactId>
  <version>2.22.2</version>
  <configuration>
    <argLine>${failsafe.jacoco.args}</argLine>
  </configuration>
  <executions>
    <execution>
      <goals>
        <goal>integration-test</goal>
        <goal>verify</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

If you run `mvn verify` you should see the following:

```txt
[INFO] --- jacoco-maven-plugin:0.8.4:prepare-agent (before-integration-test-execution) @ jacocoexample ---
[INFO] failsafe.jacoco.args set to -javaagent:/Users/nat/.m2/repository/org/jacoco/org.jacoco.agent/0.8.4/org.jacoco.agent-0.8.4-runtime.jar=destfile=/Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-integration-tests.exec
[INFO] 
[INFO] --- maven-failsafe-plugin:2.22.2:integration-test (default) @ jacocoexample ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.natritmeyer.jacocoexample.integration.IntegratedThingsIT
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.045 s - in com.natritmeyer.jacocoexample.integration.IntegratedThingsIT
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
[INFO] --- jacoco-maven-plugin:0.8.4:report (after-integration-test-execution) @ jacocoexample ---
[INFO] Loading execution data file /Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-integration-tests.exec
[INFO] Analyzed bundle 'jacocoexample' with 3 classes
```

Just like what happened for the unit tests, the first thing that happened was
the population of the `failsafe.jacoco.args` property. Next came test execution,
and finally the report was generated. The result should be an `index.html` file
in the`target/site/jacoco-integration-test-coverage-report` directory. Open it
and it should look something like this:

![Jacoco Integration Test Coverage Report](/assets/images/jacoco_integration_test_coverage_report.png)

# Merged test coverage report

We'll now configure jacoco to create a report based on both unit _and_ integration
test coverage. Achieving this will require first merging

## Merging unit and integration test coverage data

Jacoco generates reports from `.exec` files. Thus far we have two of them. If
we want jacoco to generate a report based on the data in both files we first
need to merge the data from both `.exec` files into a new `.exec` file. Here's
The `<execution>` that will make that happen:

```xml
<execution>
  <id>merge-unit-and-integration</id>
    <phase>post-integration-test</phase>
      <goals>
        <goal>merge</goal>
      </goals>
      <configuration>
      <fileSets>
        <fileSet>
          <directory>${project.build.directory}/jacoco-output/</directory>
          <includes>
            <include>*.exec</include>
          </includes>
        </fileSet>
      </fileSets>
      <destFile>${project.build.directory}/jacoco-output/merged.exec</destFile>
    </configuration>
  </execution>
<execution>
```

The `<fileSet>` tells jacoco which directory to look for input to its `merge`
goal. The `<include>` element tells jacoco which files to read in. In the case
above that's `*.exec*`. Finally, the `<destFile>` is where the merged coverage
data will be written.

## Generating the merged unit and integration test coverage report

The final step is to configure jacoco to generate a coverage report from the
`merged.exec` data we created in the previous step. We've already seen how to
do that - it's the same as generating a unit or integration test report:

```xml
<execution>
  <id>create-merged-report</id>
  <phase>post-integration-test</phase>
  <goals>
    <goal>report</goal>
  </goals>
  <configuration>
    <dataFile>${project.build.directory}/jacoco-output/merged.exec</dataFile>
    <outputDirectory>${project.reporting.outputDirectory}/jacoco-merged-test-coverage-report</outputDirectory>
  </configuration>
</execution>
```

There's an important thing to notice: the `<phase>post-integration-test</phase>`
element. Though this is the same phase as the previous step that created the
merged coverage data file, because maven respects the sequence of `<execution>`
blocks this last report-generation step will always follow the preceding
coverage-data-file-generation step.

If you run `mvn verify` now you should see the following:

```txt
[INFO] --- jacoco-maven-plugin:0.8.4:merge (merge-unit-and-integration) @ jacocoexample ---
[INFO] Loading execution data file /Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-integration-tests.exec
[INFO] Loading execution data file /Users/nat/dev/jacocoexample/target/jacoco-output/jacoco-unit-tests.exec
[INFO] Writing merged execution data to /Users/nat/dev/jacocoexample/target/jacoco-output/merged.exec
[INFO] 
[INFO] --- jacoco-maven-plugin:0.8.4:report (create-merged-report) @ jacocoexample ---
[INFO] Loading execution data file /Users/nat/dev/jacocoexample/target/jacoco-output/merged.exec
[INFO] Analyzed bundle 'jacocoexample' with 3 classes
```

The first block explains what happened in the merge, the second block tells you
that the merged coverage report was generated.

In your `target/site/jacoco-merged-test-coverage-report` directory you should
find an `index.html` file. Open it and you'll see something like the following:

![Jacoco Merged Test Coverage Report](/assets/images/jacoco_merged_test_coverage_report.png)

There you go. A test coverage report based on the execution of both unit and
integration tests.

# Failing the build

Putting aside religious wars about what level of test coverage is acceptable,
here's how to get jacoco to fail the build based on the aggregate coverage of
the unit and integration tests (as opposed to the unit and integration test
phases individually).

We'll use a jacoco `check` `<execution>` block to do the job. This example will
fail the build if coverage, taking both unit and integration tests, drops below
100%:

```xml
<execution>
  <id>check</id>
  <phase>verify</phase>
  <goals>
    <goal>check</goal>
  </goals>
  <configuration>
    <rules>
      <rule>
        <element>CLASS</element>
        <excludes>
          <exclude>*Test</exclude>
          <exclude>*IT</exclude>
        </excludes>
        <limits>
          <limit>
            <counter>LINE</counter>
            <value>COVEREDRATIO</value>
            <minimum>100%</minimum>
          </limit>
        </limits>
      </rule>
    </rules>
    <dataFile>${project.build.directory}/jacoco-output/merged.exec</dataFile>
  </configuration>
</execution>
```

Just one thing to note... the check is being executed against the `merged.exec`
coverage data file created further up by the merge `<execution>` block.

Hopefully the above makes sense and was easy enough to follow. For those who're
here just to copy and paste the whole lot into your `pom.xml`, here's what you need:

# TL;DR

If you want the whole thing in one hit, here it is:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.4</version>
      <executions>
        <execution>
        <id>before-unit-test-execution</id>
        <goals>
          <goal>prepare-agent</goal>
        </goals>
        <configuration>
          <destFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</destFile>
          <propertyName>surefire.jacoco.args</propertyName>
        </configuration>
      </execution>
      <execution>
        <id>after-unit-test-execution</id>
          <phase>test</phase>
          <goals>
            <goal>report</goal>
          </goals>
          <configuration>
            <dataFile>${project.build.directory}/jacoco-output/jacoco-unit-tests.exec</dataFile>
            <outputDirectory>${project.reporting.outputDirectory}/jacoco-unit-test-coverage-report</outputDirectory>
          </configuration>
        </execution>
        <execution>
          <id>before-integration-test-execution</id>
          <phase>pre-integration-test</phase>
          <goals>
            <goal>prepare-agent</goal>
          </goals>
          <configuration>
            <destFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</destFile>
            <propertyName>failsafe.jacoco.args</propertyName>
          </configuration>
        </execution>
        <execution>
          <id>after-integration-test-execution</id>
          <phase>post-integration-test</phase>
          <goals>
            <goal>report</goal>
          </goals>
          <configuration>
            <dataFile>${project.build.directory}/jacoco-output/jacoco-integration-tests.exec</dataFile>
            <outputDirectory>${project.reporting.outputDirectory}/jacoco-integration-test-coverage-report</outputDirectory>
          </configuration>
        </execution>
        <execution>
          <id>merge-unit-and-integration</id>
          <phase>post-integration-test</phase>
          <goals>
            <goal>merge</goal>
          </goals>
          <configuration>
            <fileSets>
              <fileSet>
                <directory>${project.build.directory}/jacoco-output/</directory>
                <includes>
                  <include>*.exec</include>
                </includes>
              </fileSet>
            </fileSets>
            <destFile>${project.build.directory}/jacoco-output/merged.exec</destFile>
          </configuration>
        </execution>
        <execution>
          <id>create-merged-report</id>
          <phase>post-integration-test</phase>
          <goals>
            <goal>report</goal>
          </goals>
          <configuration>
            <dataFile>${project.build.directory}/jacoco-output/merged.exec</dataFile>
            <outputDirectory>${project.reporting.outputDirectory}/jacoco-merged-test-coverage-report</outputDirectory>
          </configuration>
        </execution>
        <execution>
          <id>check</id>
          <phase>verify</phase>
          <goals>
            <goal>check</goal>
          </goals>
          <configuration>
            <rules>
              <rule>
                <element>CLASS</element>
                <excludes>
                  <exclude>*Test</exclude>
                  <exclude>*IT</exclude>
                </excludes>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>100%</minimum>
                  </limit>
                </limits>
              </rule>
            </rules>
            <dataFile>${project.build.directory}/jacoco-output/merged.exec</dataFile>
          </configuration>
        </execution>
      </executions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>2.22.2</version>
      <configuration>
        <argLine>${surefire.jacoco.args}</argLine>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <version>2.22.2</version>
      <configuration>
        <argLine>${failsafe.jacoco.args}</argLine>
      </configuration>
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

Remember, if you don't want low test coverage to fail the build,
remove the `check` `<execution>` block.

