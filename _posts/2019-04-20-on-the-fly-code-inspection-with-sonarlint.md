---
title:  "On-The-Fly Code Inspection with SonarLint"
excerpt: "With this quick tutorial, you’ll be able to get the IntelliJ IDE to inspect your code on-the-fly using [SonarLint](https://www.sonarlint.org/), allowing you to instantly correct the issues and only commit clean and maintainable code."
classes: wide
toc: true
categories: [code-quality]
tags: [code inspection, sonarlint, sonarqube]
---

With this quick tutorial, you’ll be able to get the IntelliJ IDE to inspect your code on-the-fly, allowing you to instantly correct the issues and only commit clean and maintainable code.

> SonarLint is an IDE extension that helps you detect and fix quality issues as you write code.
  Like a spell checker, SonarLint squiggles flaws so that they can be fixed before committing code.
    

# Use SonarLint in IntelliJ IDEA

You can start using the SonarLint analyzer by simply installing its plugin: 

{% include figure image_path="/assets/images/sonarlint/intellij_plugin.png" %} 

After restarting the IDE, you can instantly benefit from on-the-fly code inspections when coding in one of the supported languages: Java, JavaScript, Python, Kotlin, Ruby or PHP. 

{% include figure image_path="/assets/images/sonarlint/java_inspection.png" %} 

# Integration with SonarQube

Unfortunately, by default SonarLint has no checking rules for Scala source code, however it supports an amazing feature of connecting to an external SonarQube to have server rules, issues and exclusions synched.

Before binding SonarLint to your SonarQube server, you need to configure your project inspection on SonarQube. To do so, follow the steps presented in my last [post](/code-quality/local-code-inspection-with-sonarqube/) about SonarQube. 

In IntelliJ preferences, first connect to a server via the *SonarLint General Settings*, then bind the project under *SonarLint Project Settings*:

{% include figure image_path="/assets/images/sonarlint/sonarqube_config2.png" %}

{% include figure image_path="/assets/images/sonarlint/sonarqube_config.png" %} 

Immediately after binding, SonarLint will also check your Scala code.

{% include figure image_path="/assets/images/sonarlint/scala_inspection.png" %} 

  