---
title:  "Dealing with Maven conflicts"
excerpt: "In this post I'm explaining the main concepts behind Maven dependency management, conflict detection and its resolution."
classes: wide
categories: [software-development]
tags: [maven, mvn, shading]
---

TODO...

If you are working with JVM-based Maven projects sooner or later you will end-up frustrated with your application crashing in runtime with NoSuchMethodError, NoClassDefFoundError or ClassNotFoundException, commonly known as [Dependency hell](https://en.wikipedia.org/wiki/Dependency_hell)

This happens if you add to your project dependencies already somewhere else declared or included transitively by another libraries, if their version and implementation don't match.


### Conflict detection

[Maven Dependency Plugin](https://maven.apache.org/plugins/maven-dependency-plugin/analyze-mojo.html)

[Maven Enforcer Plugin](https://maven.apache.org/enforcer/maven-enforcer-plugin/)

[Maven Helper for IntelliJ](https://plugins.jetbrains.com/plugin/7179-maven-helper)

[Java API Compliance Checker (JAPICC)](https://github.com/lvc/japi-compliance-checker)


