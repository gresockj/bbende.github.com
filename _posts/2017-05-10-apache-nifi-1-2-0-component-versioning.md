---
layout: post
title: "Apache NiFi - Component Versioning"
description: ""
category: "Development"
tags: [NiFi]
---
{% include JB/setup %}

One of the new features in Apache NiFi 1.2.0 is the ability to understand versions of components
(processor, controller service, reporting task) and run different versions at the same time, with the
ability to change between versions.

This feature is generally referred to as "component versioning" or "multiple versions of the same component". The rest of
this post will take a look at how component versioning is implemented and why it is important to future capabilities.

### Versioning NARs

As a quick refresher, NiFi extensions are generally packaged as NiFi ARchives (NARs). A NAR contains the code for one or
more extensions, as well as the dependencies required by those extensions. During start-up NiFi unpacks the NARs and uses
Java's ServiceLoader to identify the available extensions. Each NAR is given its own class loader to isolate
dependencies across NARs, and each NAR can have a dependency on one parent NAR.

The standard way to create a NAR is by using the *NAR Maven Plugin* provided by the NiFi community. As part of creating the NAR,
the plugin generates a MANIFEST file that ends up in the META-INF directory of the NAR. Looking at the MANIFEST file of
nifi-hadoop-nar from a previous version of NiFi, you would see something like the following:

    Manifest-Version: 1.0
    Archiver-Version: Plexus Archiver
    Built-By: bbende
    Nar-Id: nifi-hadoop-nar
    Nar-Dependency-Id: nifi-hadoop-libraries-nar
    Created-By: Apache Maven 3.3.3
    Build-Jdk: 1.8.0_74

The *Nar-Id* entry is the id of the current NAR, and the *Nar-Dependency-Id* is the id of another NAR that the current NAR is
dependent on, in this case nifi-hadoop-libraries-nar.

Prior to releasing NiFi 1.2.0, the NAR Maven Plugin was updated to provide more information in the MANIFEST file.
Specifically, the MANIFEST now contains entries for the version and group of the current NAR, the version and group of
the dependent NAR (if one exists), and other build information. Looking at the MANIFEST of nifi-hadoop-nar from the 1.2.0
release, you would see the following:

    Manifest-Version: 1.0
    Archiver-Version: Plexus Archiver
    Built-By: bbende
    Nar-Group: org.apache.nifi
    Nar-Id: nifi-hadoop-nar
    Nar-Version: 1.2.0
    Nar-Dependency-Group: org.apache.nifi
    Nar-Dependency-Id: nifi-hadoop-libraries-nar
    Nar-Dependency-Version: 1.2.0
    Build-Tag: nifi-1.2.0-RC2
    Build-Revision: f4f174b
    Build-Branch: NIFI-3770-RC2
    Build-Timestamp: 2017-05-05T20:39:20Z
    Created-By: Apache Maven 3.3.3
    Build-Jdk: 1.8.0_74

If you have any NARs that haven't been rebuilt with the latest NAR Maven plugin, they will still work with NiFi 1.2.0,
but will be given a version of "unversioned" and a group of "default".

*NOTE: The term "bundle" will be used to refer to where a component came from. In most cases a bundle is a NAR, but
technically components with no dependencies can be deployed to NiFi's lib directory in a JAR. The "system" bundle is a
logical bundle that represents all JARs directly in the lib directory.*

### Creating Components

One of the places where component versioning is noticeable is adding components. The screens for
adding processors, controller services, and reporting tasks now show a "Version" column, as well as a "Source" menu
to select the group.

<img src="{{ BASE_PATH }}/assets/images/nifi-component-versioning/01-add-processor.png" class="img-responsive">

As an example, lets say someone has created two custom NARs that both contain org.apache.nifi.MyProcessor. Each NAR has
the same Nar-Group and Nar-Id, but NAR #1 has a Nar-Version of 1.0 and NAR #2 has a Nar-Version of 2.0. This results in
two versions of MyProcessor showing up in the list:

<img src="{{ BASE_PATH }}/assets/images/nifi-component-versioning/02-add-processor-my-processor.png" class="img-responsive">

If we add MyProcessor 1.0 and 2.0 to the flow, we can see that the processors now indicate the bundle
information.

<img src="{{ BASE_PATH }}/assets/images/nifi-component-versioning/03-my-processor-v1-and-v2.png" class="img-responsive">

### Changing Versions

We can also change the version of a component in-place by right-clicking on the processor and selecting "Change Version":

<img src="{{ BASE_PATH }}/assets/images/nifi-component-versioning/04-my-processor-change-version.png" class="img-responsive">

*NOTE: You must stop a component before changing versions, and the "Change Version" menu option will only display if there
are other versions to change to.*

Selecting "Change Version" will display prompt with the other available versions:

<img src="{{ BASE_PATH }}/assets/images/nifi-component-versioning/05-my-processor-change-version-2.png" class="img-responsive">

Changing versions is only allowed across bundle's with the same group and id.

For example, MyProcessor 1.0 and 2.0 both come from bundles with a group of org.apache.nifi and an id of nifi-example-processors,
so changing versions between 1.0 and 2.0 is allowed. If there was another bundle that also had MyProcessor, but had a
different group or id, then it wouldn't be available to change to.

### Storing Bundle Info

Behind the scenes the bundle information for a component is serialized to the flow.xml.gz:

      <class>org.apache.nifi.processors.example.MyProcessor</class>
      <bundle>
        <group>org.apache.nifi</group>
        <artifact>nifi-example-processors-nar</artifact>
        <version>1.0</version>
      </bundle>

When NiFi loads the flow during start up, it finds the class and bundle information for each component. If a
component doesn't have bundle information, or if it has a bundle that doesn't exist, then NiFi will find bundles that
contain the class name of the given component. If there is only one bundle containing that class name then that becomes
the bundle for the component.

This is how flows created prior to 1.2.0 will automatically select the appropriate bundle, and how upgrading to future
versions will select the appropriate bundle when all the bundles change from 1.2.0 to 1.3.0 for example.

### Why is Component Versioning Important?

While running multiple versions of the same component at the same time is cool, the concept of component versioning also
plays a significant role in future capabilities, such as the [flow registry](https://cwiki.apache.org/confluence/display/NIFI/Configuration+Management+of+Flows) and [extension registry](https://cwiki.apache.org/confluence/display/NIFI/Extension+Repositories+%28aka+Extension+Registry%29+for+Dynamically-loaded+Extensions).

The Flow Registry refers to improving the software development life-cycle around building flows and deploying them across
environments. In order to version control a flow, it is important to capture the versions of the components the flow was
running. If a flow is deployed from a dev environment to a prod environment, and dev is running version 1.0 of a processor
and prod is running version 2.0, the flow may not produce the same results in prod as it did in dev.

The Extension Registry refers to creating a place for publishing and discovering NARs, so that every NAR doesn't need to be
bundled with NiFi by default. Many different groups may want to publish their own variation of a processor such as PutHDFS,
and the consumers must be able to understand which version they are pulling in. In addition, when deploying a versioned flow
from the flow registry, the target environment may not have all of the components in the flow, and the extension registry
provides a central place of these components to exist.

So while component versioning may not seem like a big deal right now, it will play a significant role in the future to come.
