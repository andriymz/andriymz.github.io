---
title:  "Local Code Inspection with SonarQube"
excerpt: "With this quick tutorial, you'll be able to inspect your code, written in both Scala or Java, locally with SonarQube running in a Docker container."
classes: wide
toc: true
categories: [code-quality]
tags: [code inspection, sonarqube]
---

With this quick tutorial, you'll be able to inspect your code, written in both Scala or Java, locally with [SonarQube](https://www.sonarqube.org) running in a Docker container.

> SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities on 20+ programming languages. SonarQube offers reports on duplicated code, coding standards, unit tests, code coverage, code complexity, comments, bugs, and security vulnerabilities.
> --Wiki

# SonarQube Docker container

There is an official image of SonarQube you can download from [Docker Hub](https://hub.docker.com/_/sonarqube):

```
docker pull sonarqube
```

After downloading, you need to start the container by running:

```
docker run -d --name sonarqube -p 9000:9000 sonarqube
```

SonarQube becomes available at [http://localhost:9000](). Use **admin** as _username_ and _password_ to login.

{% include figure image_path="/assets/images/sonarqube/home_page.png" %}


# Project Creation

After logging in, click on *Create new project* button.

{% include figure image_path="/assets/images/sonarqube/home_page_after_login.png" %}

Fill the *Project key* with `groupId:artifactId` of your Maven project, and provide a desired *Display name*.

{% include figure image_path="/assets/images/sonarqube/create_project_1.png" %}


Then click on *Generate button* to get an identification **token**.

{% include figure image_path="/assets/images/sonarqube/create_project_2.png" %}


Note this **token** somewhere, since you will need it later.

{% include figure image_path="/assets/images/sonarqube/create_project_3.png" %}


# SonarQube Maven Plugin

To prepare your project to be analyzed, you only need to include the Maven plugin below (see [Analyzing with SonarQube Scanner for Maven](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Maven)):

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.6.0.1398</version>
</plugin>
```

To run the analysis execute the `sonar:sonar` goal by specifying the registered *projectKey*, the host where SonarQube runs, the generated **token**, the folder where Java and Scala source directories reside and their versions, as follows:

```bash
mvn sonar:sonar \
-Dsonar.projectKey=com.xpandit.bdu:testing-sonarqube \
-Dsonar.host.url=http://localhost:9000 \
-Dsonar.login=b837afb95e5d8703e7cdaa8cba0ce6ff79fb35ea \
-Dsonar.sources=src/main/ \
-Dsonar.java.version=1.8 \
-Dsonar.scala.version=2.11
```

A successful build provides the dashboard *url* at the end:

```bash
...

[INFO] Analysis report generated in 96ms, dir size=79 KB
[INFO] Analysis report compressed in 13ms, zip size=13 KB
[INFO] Analysis report uploaded in 64ms
[INFO] ANALYSIS SUCCESSFUL, you can browse http://localhost:9000/dashboard?id=com.xpandit.bdu%3Atesting-sonarqube
[INFO] Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
[INFO] More about the report processing at http://localhost:9000/api/ce/task?id=AWoMjswXDM5J3fj3PeoZ
[INFO] Analysis total time: 8.755 s
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 12.073 s
[INFO] Finished at: 2019-04-11T14:20:46+01:00
[INFO] Final Memory: 50M/673M
[INFO] ------------------------------------------------------------------------
```


# Inspection Result

Once the project builds successfully, you can check the inspection results by navigating to the provided dashboard *url* or at any time to the *Projects* section.

{% include figure image_path="/assets/images/sonarqube/projects_after_build.png" %}


The test project I analyzed has 8 code smells in total from `TestMyJava` and `TestMyScala` classes:

{% include figure image_path="/assets/images/sonarqube/code_smells.png" %}


# Inspection Rules

You can check the complete list of rules considered when analyzing your Scala or Java code for bugs, code smells and vulnerabilities.

[https://www.sonarsource.com/products/codeanalyzers/sonarjava.html](https://www.sonarsource.com/products/codeanalyzers/sonarjava.html)

[https://www.sonarsource.com/products/codeanalyzers/sonarscala.html](https://www.sonarsource.com/products/codeanalyzers/sonarjava.html)