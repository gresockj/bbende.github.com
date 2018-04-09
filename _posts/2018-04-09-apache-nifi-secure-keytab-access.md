---
layout: post
title: "Apache NiFi - Secure Keytab Access"
description: ""
category: "Development"
tags: [NiFi,Kerberos]
---
{% include JB/setup %}

In a multi-tenant environment, a typical approach is to create a process group per team and apply security
policies that restrict access, such that a given team only has access to it's own group.

Within a process group, a team may need to interact with an external system that is secured with Kerberos. As an
example, lets say there is a Kerberized HDFS cluster, and there are two teams that each have a keytab to
access this cluster.

In order to create a PutHDFS processor that sends data to the Kerberized HDFS cluster, the processor must be configured
with a principal and keyab, and the keytab must be on a filesystem that is accessible to the NiFi JVM. In addition, the keytab must be readable by the operating system user that launched the NiFi JVM.

<img src="{{ BASE_PATH }}/assets/images/nifi-keytab-isolation/01-nifi-keytab-setup-before.png" class="img-responsive">

The problem here is that the Keytab property is a free-form text field, so there is no way to stop team 1 from entering team 2's keytab.

In order to solve this problem, the 1.6.0 release introduced a Keytab Controller Service, along with more granular restricted components, which together can be used to properly secure access to keytabs.

### Keytab Controller Service

The first step to securing keytab access was to create a controller service where the principal and keytab could be
defined, and then update all processors that had the free-form properties to also have a property for a Keytab
Service.

NOTE: The free-form properties were not removed on this release in order to give everyone a chance to migrate to
the new approach, but we'll discuss this more later.

The benefit of using a controller service is that we can restrict which users have ability to use the service via security policies. In our example above, an admin user can create a Keytab Service for team 1 that points to team 1's keytab, and then create a policy that gives only team 1 access to this service. The admin user can then do the same thing for team 2.  

<img src="{{ BASE_PATH }}/assets/images/nifi-keytab-isolation/02-nifi-keytab-setup-after.png" class="img-responsive">

This gets us very close to what we want, BUT *what if someone from team 1 ignores the service that was created for them, and just creates another Keytab Service pointing to team 2's keytab*?

We need a way to also restrict who can create Keytab Controller services, so that only the admin users can create them,
and not any of the users in team 1 or team 2.

### Granular Restricted Components

The NiFi 1.0.0 release introduced the concept of "restricted components". The idea was that some components may be
able to do things that we don't want all users to do, such as access the local file system, or execute code. These types of components required a user to have an additional "restricted" permission in order to create the component.

The 1.6.0 release introduced more granular categories of restricted components:

* read-filesystem
* write-filesystem
* execute-code
* access-keytab
* export-nifi-details

The restricted permissions are controlled from the global policies menu in the top-right:

<img src="{{ BASE_PATH }}/assets/images/nifi-keytab-isolation/03-nifi-restricted-permissions.png" class="img-responsive img-thumbnail">

In our previous example, if we only give the admin users the "access-keytab" permission, then the users in team 1 and team 2 won't be able to create a Keytab Controller service, thus forcing them to use the Keytab Services they were given permission to.

### Preventing Free-form Keytab Properties

The final piece to the puzzle is to prevent the use of the old free-form keytab properties that were left around for
backwards compatibility. This can be done by configuring an environment variable in nifi-env.sh:

    export NIFI_ALLOW_EXPLICIT_KEYTAB=true

Setting this value to *false* will produce a validation error in any component where the free-form keytab property is
entered, which means the component can't be started unless it uses a Keytab Controller service.

### Summary

In summary...

* Only admin users have the "access-keytab" permission
* Admin users create Keytab Services and assign appropriate permissions
* Regular users can use the Keytab Services they were given access to, but can't create new ones
* Legacy free-form properties are disallowed via the NIFI_ALLOW_EXPLICIT_KEYTAB environment variable
