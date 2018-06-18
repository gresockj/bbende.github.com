---
layout: post
title: "Apache NiFi Registry 0.2.0"
description: ""
category: "Development"
tags: [NiFi]
---
{% include JB/setup %}

This post will introduce some of the new features in the 0.2.0 release of NiFi Registry, including git-based flow
persistence, a more configurable metadata database, and event hooks.

### Initial Architecture

The initial 0.1.0 release provided the following architecture, primarily centered around the metadata database and the
flow persistence provider:

<img src="{{ BASE_PATH }}/assets/images/nifi-registry-0_2_0/01-architecture-original.png" class="img-responsive">

The *metadata database* stores all the information about buckets and versioned items, such as identifiers, names, and
descriptions, as well as which items belong to which bucket. This component is considered an internal component, which
means it must be a relational database, and can't be switched to another storage mechanism.

The *flow persistence provider* stores the content of each versioned flow. This component is a public extension
point, which means custom implementations can be provided by implementing the FlowPersistenceProvider interface. The
Java ServiceLoader will be used to scan the classpath for all available implementations during start-up.
Implementations can be provided by adding a JAR to the lib directory, or by utilizing a custom extension directory
containing one or more JARs (documented in nifi-registry.properties).

The overall idea is that the metadata database contains information across all types of versioned items, which may eventually
be more than just flows. Each type of item may then have its own persistence mechanism for the content of the item.

### Git FlowPersistenceProvider

The 0.2.0 release provides a new implementation of the FlowPersistenceProvider which utilizes the JGit library to interact
with a git repository. This means the content of the versioned flows can now be stored in git.

<img src="{{ BASE_PATH }}/assets/images/nifi-registry-0_2_0/02-architecture-git.png" class="img-responsive">

The git provider can be configured in providers.xml via the following configuration:

    <flowPersistenceProvider>
        <class>
          org.apache.nifi.registry.provider.flow.git.GitFlowPersistenceProvider
        </class>
        <property name="Flow Storage Directory">./flow_storage</property>
        <property name="Remote To Push"></property>
        <property name="Remote Access User"></property>
        <property name="Remote Access Password"></property>
    </flowPersistenceProvider>

The "Flow Storage Directory" property specifies a local directory that is expected to already be a git repository. This
could be done by creating a new directory and running "git init", or from cloning an existing remote repository.

The "Remote To Push" property specifies the name of remote to automatically push to. This property is optional and if not
specified, then commits will remain in the local repository, unless a push is performed manually.

NOTE: In order to use GitHub, do the following:

* Create a new GitHub repo and clone it locally
* Set "Flow Storage Directory" to the local directory where the repo was cloned
* Go to GitHub's "Developer Settings" for your account
* Create a new "Person Access Token" for your NiFi Registry instance
* Set your GitHub username as the "Remote Access User"
* Set the access token as the "Remote Access Password"

### External Metadata Database

The 0.2.0 release provides new configuration that allows the metadata database to utilize an external database.
Currently Postgres is the only supported database besides H2, although others may work with additional testing.

<img src="{{ BASE_PATH }}/assets/images/nifi-registry-0_2_0/03-architecture-postgres.png" class="img-responsive">

The 0.1.0 release had two properties in nifi-registry.properties related to the H2 database:

    nifi.registry.db.directory=
    nifi.registry.db.url.append=

The 0.2.0 release adds the following properties:

    nifi.registry.db.url=
    nifi.registry.db.driver.class=
    nifi.registry.db.driver.directory=
    nifi.registry.db.username=
    nifi.registry.db.password=

When the application first starts, it will determine if the old nifi.registry.db.directory property is populated,
and if so it will attempt to locate an existing H2 database. If an existing database is found, and if the target database
is empty, then data will automatically be migrated to the new database.

For specifics about each property please refer to the NiFi Registry Administration Guide.

### Event Hooks

An event hook is a new extension point that allows custom code to be executed when specific application events occur.

In order to implement an event hook, the following Java interface must be implemented:

    public interface EventHookProvider extends Provider {

        void handle(Event event) throws EventHookException;

    }

The Event object will contain an EventType, along with a list of field/value pairs that are specific to the event. At the
time of writing this, the possible event types are the following:

    CREATE_BUCKET
    CREATE_FLOW
    CREATE_FLOW_VERSION
    UPDATE_BUCKET
    UPDATE_FLOW
    DELETE_BUCKET
    DELETE_FLOW

The list of event types and fields can be checked [here](https://github.com/apache/nifi-registry/blob/master/nifi-registry-provider-api/src/main/java/org/apache/nifi/registry/hook/EventType.java).

Similar to the FlowPersistenceProvider, EventHookProviders will be discovered via Java's ServiceLoader scanning the classpath
during start-up.

There are two EventHookProviders that come with the 0.2.0 release, the *LoggingEventHookProvider* and the *ScriptedEventHookProvider*.

#### LoggingEventHookProvider

The *LoggingEventHookProvider* logs a string representation of each event using an SLF4J logger which can be configured via
NiFi Registry's logback.xml.

By default there is an appender named "EVENTS_FILE" which appends to a log file named nifi-registry-event.log in the logs directory.

To enable this hook, simply uncomment the configuration in providers.xml:

    <eventHookProvider>
        <class>
          org.apache.nifi.registry.provider.hook.LoggingEventHookProvider
        </class>
    </eventHookProvider>  

#### ScriptEventHookProvider

The *ScriptedEventHookProvider* allows a custom script to be executed for each event.

The configuration for this event hook looks like the following:

    <eventHookProvider>
        <class>
          org.apache.nifi.registry.provider.hook.ScriptEventHookProvider
        </class>
        <property name="Script Path"></property>
        <property name="Working Directory"></property>
    </eventHookProvider>

The "Script Path" property is the full path to a script that will executed for each event. The arguments to the script will be
the event fields in the order they are specified for the event type.

Example use-cases for this might be to send out a notification when a flow version is created, or execute
commands from the NiFi CLI to automatically deploy a flow when a new version is created.

### Summary

With the ability to automatically push versioned flows to a remote git repository, as well as leveraging an external
Postgres database, all data can be off-loaded from the server where NiFi Registry is running, making it easy
to start up a new instance of the application in the event some type of failure occurs. In addition, the event hooks
provide a configurable way to facilitate custom workflows around specific actions being taken in the registry.
