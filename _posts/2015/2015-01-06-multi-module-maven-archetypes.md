---
layout: post
title: "Multi-Module Maven Archetypes"
description: ""
category: "Development"
tags: [Maven,Archetype]
---
{% include JB/setup %}

A Maven Archetype is a useful way to create a template for new projects. There are many publicly available 
archetypes, probably the most well known being the maven-archetype-quickstart, but you can also create your 
own. 

Since many projects have multiple child modules, how do you create an archetype for that kind of project?

The trick is to use "\_\_rootArtifactId\_\_" in the module directory names. Maven will replace this with the 
overall artifactId you entered when running archetype:generate, the same way it will replace ${artifactId} in pom files.

An example structure for a multi-module archetype is the following:

    example-archetype
    ├── pom.xml
    └── src
        ├── main
           └── resources
               ├── META-INF
               │   └── maven
               │       └── archetype-metadata.xml
               └── archetype-resources
                   ├── __rootArtifactId__-module1
                   │   ├── pom.xml
                   │   └── src
                   │       ├── main
                   │       │   ├── java
                   │       │   └── resources
                   │       └── test
                   │           └── java
                   │           └── resources
                   ├── __rootArtifactId__-module2
                   │   ├── pom.xml
                   │   └── src
                   │       ├── main
                   │       │   ├── java
                   │       │   └── resources
                   │       └── test
                   │           └── java
                   │           └── resources
                   └── pom.xml

The structure under archetype-resources is what will become the new project during archetype:generate. 

#### Example Parent Pom

The pom directly under archetype-resources will be the parent pom of the project that gets created:

    <groupId>${groupId}</groupId>
    <artifactId>${artifactId}</artifactId>
    <version>${version}</version>
    <packaging>pom</packaging>

    <modules>
        <module>${rootArtifactId}-module1</module>
        <module>${rootArtifactId}-module2</module>
    </modules>
    
#### Example Module1 Pom

The child modules can reference the parent with ${rootArtifactId}:

    <parent>
        <groupId>${groupId}</groupId>
        <artifactId>${rootArtifactId}</artifactId>
        <version>${version}</version>
    </parent>

    <artifactId>${artifactId}</artifactId>
    <packaging>jar</packaging>
    
#### Example Module2 Pom

Assuming module2 depends on module1:

    <parent>
        <groupId>${groupId}</groupId>
        <artifactId>${rootArtifactId}</artifactId>
        <version>${version}</version>
    </parent>

    <artifactId>${artifactId}</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>${groupId}</groupId>
            <artifactId>${rootArtifactId}-module1</artifactId>
            <version>${version}</version>
        </dependency>
    </dependencies>

#### Example archetype-metadata.xml

An example archetype-metadata.xml is below, a full description of the available options can be 
found in the [Archetype Documentation.](http://maven.apache.org/archetype/archetype-common/archetype-descriptor.html)

    <archetype-descriptor name="${artifactId}">
      <modules>
        <module id="${rootArtifactId}-module1" dir="__rootArtifactId__-module1" name="${rootArtifactId}-module1">
            <fileSets>
                <fileSet filtered="true" packaged="true" encoding="UTF-8">
                    <directory>src/main/java</directory>
                </fileSet>
                <fileSet encoding="UTF-8">
                    <directory>src/main/resources</directory>
                </fileSet>
                <fileSet filtered="true" packaged="true" encoding="UTF-8">
                    <directory>src/test/java</directory>
                </fileSet>
                 <fileSet encoding="UTF-8">
                    <directory>src/test/resources</directory>
                 </fileSet>
            </fileSets>
        </module>
        <module id="${rootArtifactId}-module2" dir="__rootArtifactId__-module2" name="${rootArtifactId}-module2">
            <fileSets>
                <fileSet filtered="true" packaged="true" encoding="UTF-8">
                    <directory>src/main/java</directory>
                </fileSet>
                <fileSet encoding="UTF-8">
                    <directory>src/main/resources</directory>
                </fileSet>
                <fileSet filtered="true" packaged="true" encoding="UTF-8">
                    <directory>src/test/java</directory>
                </fileSet>
                 <fileSet encoding="UTF-8">
                    <directory>src/test/resources</directory>
                 </fileSet>
            </fileSets>
        </module>
      </modules>
    </archetype-descriptor>
    
#### Links

The following links were helpful in learning how to create this example:

* [http://maven.apache.org/archetype/archetype-common/archetype-descriptor.html](http://maven.apache.org/archetype/archetype-common/archetype-descriptor.html)

* [http://gsmsengupta.blogspot.com/2014/01/creating-archetype-for-multi-project.html](http://gsmsengupta.blogspot.com/2014/01/creating-archetype-for-multi-project.html)

* [http://www.javacodegeeks.com/2012/02/maven-archetype-creation-tips.html](http://www.javacodegeeks.com/2012/02/maven-archetype-creation-tips.html)
