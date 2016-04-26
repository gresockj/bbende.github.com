---
layout: post
title: "Building a Plugin for Apache Ranger"
description: ""
category: "Development"
tags: [Ranger, Hadoop]
---
{% include JB/setup %}

Apache Ranger is a framework for providing centralized security administration
across the Hadoop ecosystem. Ranger provides a central location for defining security
policies which can then be used by applications for making authorization decisions.
Applications integrate with Ranger through a plugin model. The rest of this post will
describe the details of creating Ranger plugin.

### Overview

A Ranger plugin is composed of three main components:

 * **Service Definition** - A JSON document defining the service, including the types
 of resources, and the possible actions on those resources.
 * **Resource Lookup Service** - A Java service used by Ranger to retrieve available
 resources from the application being secured.
 * **Authorization Plugin** - A Java plugin that runs inside the application being secured
 to make authorization decisions based on policies retrieved from Ranger. 

At a high level, the interaction between an application and Ranger looks like the
following:

<img src="{{ BASE_PATH }}/assets/images/ranger-plugins/ranger-overview.png" class="img-responsive">

First, the application's service definition and resource lookup code would
be installed in Ranger (more on this later). An administrative user would then use
the Ranger Admin UI to define policies for the given application. A policy gives
one or more users permission to perform specific actions on a resource. A resource
is anything the application wants to restrict access to, such as a database table
or column, a directory, an index, etc.

The authorization plugin would be deployed in the application being secured, and
periodically retrieves the relevant policies from Ranger in a background process.
When a user makes a request to the application, the application would compare the
resource being accessed, along with the user making the request, against the retrieved
policies in order to determine access.

### Service Definition

### Client

### Plugin
