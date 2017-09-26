---
layout: post
title: "Apache Ranger - Building a Plugin"
description: ""
category: "Development"
tags: [Ranger, Hadoop]
---
{% include JB/setup %}

Apache Ranger is a framework for providing centralized security administration
across the Hadoop ecosystem. Ranger provides a central location for defining security
policies that can be used by other applications for making authorization decisions.
Applications integrate with Ranger through a standard plugin model. The rest of this
post will describe the details of creating a Ranger plugin.

### Overview

In order for an application to integrate with Ranger, there are three components involved:

 * **Service Definition** - A JSON document defining the service, including the types
 of resources, and the possible actions on those resources.
 * **Resource Lookup Service** - A Java service used by Ranger to retrieve available
 resources from the application being secured.
 * **Authorizer** - A Java service that runs inside the application being secured
 to make authorization decisions based on policies retrieved from Ranger.

At a high level, the interaction between an application and Ranger looks like the
following:

<img src="{{ BASE_PATH }}/assets/images/ranger-plugins/ranger-overview.png" class="img-responsive">

First, the application's service definition and resource lookup code is
installed in Ranger (more on this later). An administrative user then uses
the Ranger Admin UI to define policies for the given application. A policy gives
one or more users permission to perform specific actions on a resource. A resource
is anything the application wants to restrict access to, such as a database table
or column, a directory, an index, etc.

The Ranger authorizer is deployed in the application being secured, and periodically retrieves
the relevant policies from Ranger in a background process. When a user makes a request to the
application, the application delegates to the authorizer to determine if the user making the
request has permission to perform the request action on the given resource.

In addition to making authorization decisions, the Ranger authorizer can also audit
the results of those decisions. The authorizer will send these audit logs to a configured
destination in a background process, in order to not interfere with the application's main
execution path. Currently Solr is the preferred destination for audit logs.

### Service Definition

An shell service definition looks like the following:

    {
      "id":10,
      "name":"example",
      "implClass":"org.apache.ranger.services.RangerExampleService",
      "label":"Example",
      "description":"Example",
      "resources":[ ],
      "accessTypes":[ ],
      "configs":[ ],
      "enums": [ ],
      "contextEnrichers":[ ],
      "policyConditions":[ ]
    }

Some notes about the fields of the service definition:

* implClass is the fully-qualified class name of a class that extends RangerBaseService
and can be used to validate the configuration and perform resource lookups.
* resources is where the types of resources for the given application are defined
* accessTypes is where the possible actions on those resources is defined
* configs is where fields for user input are defined

### Resource Lookup

The resource lookup service is the class specified by implClass in the service definition, and
must extend RangerBaseService. The following methods must be implemented:

    public abstract HashMap<String, Object> validateConfig()
      throws Exception;

    public abstract List<String> lookupResource(ResourceLookupContext context)
      throws Exception;

The validateConfig method lets the service validate any user input provided for the "configs" element
from the service definition. The lookupResource method is called when a user starts typing a resource
name in the Ranger Admin UI. This method can then call the given application to assist in
auto-completing the resource name.

### Service Installation

The service definition and resource lookup can be added to the Ranger codebase as part of the
default services bundled with Ranger, or can be added to a Ranger installation after the fact.

For services that are added to the Ranger codebase, the following steps would be taken:

* Add the service definition to [agents-common/src/main/resources/service-defs](https://github.com/apache/incubator-ranger/tree/master/agents-common/src/main/resources/service-defs)
* Modify [EmbeddedServiceDefsUtil.java](https://github.com/apache/incubator-ranger/blob/master/agents-common/src/main/java/org/apache/ranger/plugin/store/EmbeddedServiceDefsUtil.java)
to add your new service
* Create a new Maven module at the root of Ranger's source tree and place your resource lookup service
in this module
* Modify [src/main/assembly/admin-web.xml](https://github.com/apache/incubator-ranger/blob/master/src/main/assembly/admin-web.xml) to include your new module in the assembly for the Ranger Admin Webapp

For services that live outside Ranger, they can be added to a Ranger installation through the following steps:

* Make a directory *ranger-0.5.0-admin/ews/webapp/WEB-INF/classes/ranger-plugins/example* and copy the
jar with your resource lookup code to this location (replacing example with your service name)

* Post the service definition to the Ranger Admin REST endpoint:

      curl -u admin:admin -X POST -H "Accept: application/json" -H "Content-Type: application/json" --data @ranger-servicedef-myservice.json http://localhost:6080/service/plugins/definitions

* Restart the Ranger Admin service

### Authorizer

The authorizer is the code that runs in the application being secured, and communicates with
Apache Ranger to retrieve policies. Integrating the authorizer into the application is specific to each
application, but the authorizer will delegate to RangerBasePlugin through something like the
following:

    RangerAccessResourceImpl resource = new RangerAccessResourceImpl();
    resource.setValue("example-resource", identifier);

    RangerAccessRequestImpl request = new RangerAccessRequestImpl();
    request.setResource(resource);
    request.setAction(...);
    request.setAccessType(...);
    request.setUser(...);
    request.setAccessTime(...);

    RangerAccessResult result = basePlugin.isAccessAllowed(
      rangerRequest, resultProcessor);

    if (result.getIsAllowed()) {
      ...
    } else {

    }

First, a resource is created representing the resource being requested through the application. When
setValue is called on the resource, the first parameter is the type of resource (one of the types in the
service definition), and the value is the id of the resource.

Next, a RangerAccessRequest is created for the resource, including the user making the request, and the action
being performed. The request is then passed to the isAccessAllowed method of RangerBasePlugin. RangerBasePlugin
determines if there is a policy granting the user permission to perform the given action on the given
resource. The result is then checked to determine if access is allowed.

In order to know how to communicate with Ranger, and how to store audit information, RangerBasePlugin requires
two configuration files to be on the classpath:

* ranger-yourservice-security.xml
* ranger-yourservice-audit.xml

Examples of these files can be found in the Ranger code.

### Auditing

Auditing is performed by the ResultProcessor used by the RangerBasePlugin. A default result processor can
be set on the base plugin, or a result processor can be passed as a parameter to the isAccessAllowed method.
The standard result processor that performs auditing is RangerDefaultAuditHandler. If no result processor is
specified, then no auditing will be performed.

The destination of the audit information is specified in the audit XML config file described above. The configuration
required to send audit logs to Solr is the following:

    <property>
      <name>xasecure.audit.destination.solr</name>
      <value>true</value>
    </property>

    <property>
      <name>xasecure.audit.destination.solr.batch.filespool.dir</name>
      <value>/tmp/audit/solr/spool</value>
    </property>

    <property>
      <name>xasecure.audit.destination.solr.urls</name>
      <value>http://localhost:6083/solr/ranger_audits</value>
    </property>

Information on setting up Solr for Ranger can be found in [security-admin/contrib/solr_for_audit_setup](https://github.com/apache/incubator-ranger/tree/master/security-admin/contrib/solr_for_audit_setup). For an easy way to run Ranger and Solr locally, this [Vagrant project](https://github.com/bbende/apache-ranger-vagrant) can be
used.
