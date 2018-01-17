---
layout: post
title: "Apache NiFi - How do I deploy my flow?"
description: ""
category: "Development"
tags: [NiFi, Solr]
---
{% include JB/setup %}

One of the most frequently asked questions about NiFi has been *"How do I deploy my flow?"*.

A core feature of NiFi is that you can modify the live data flow without having to perform the traditional design and deploy steps. As a result, the idea of "deploying a flow" wasn't really baked into the system from the beginning.

Over time there were some attempts at solving the problem, such as putting the flow.xml.gz under version control, or scripting the deployment of templates, but each of these had it's own issues and difficulties.

In order to solve the problem, the community put a significant amount of effort into building first class features around version control of data flows. The result of this effort was the creation of a whole new Apache NiFi sub-project, called *NiFi Registry*.

NiFi Registry is a complementary application that provides a central location for storage and management of shared resources. The first release focuses on storing and managing "versioned flows", with the 1.5.0 release of NiFi adding the ability to interact with registry instances.

The rest of this post is going to use an example scenario to show how NiFi and NiFi Registry can be used to deploy a data flow.

### Example Scenario

For this example, we are going to use data from [https://randomuser.me](https://randomuser.me) which offers a nice way to generate some test data resembling a users table from a database.

To generate some sample data we can run the following curl command:

    curl "https://randomuser.me/api/?format=csv&results=10&inc=name,email,registered&nat=us" > users-10.csv

Looking at this data we should see something like:

    name.title,name.first,name.last,email,registered
    ms,lily,sims,lily.sims@example.com,2010-10-13 04:37:29
    ms,marion,elliott,marion.elliott@example.com,2009-06-14 16:40:00
    ...

For the sake of this example, lets say we have the following requirements:

* Use NiFi to ingest this data to Solr
* Convert CSV data to JSON
* Add a full_name field with combination of first name + last name
* Add a gender field based on the title
* Ingest to different Solr collections depending on environment   

To simulate this scenario we are going to setup two NiFi instances to represent DEV and PROD environments, a shared NiFi Registry, and Solr with DEV and PROD collections.

### Initial Setup

* Download artifacts

  * [NiFi 1.5.0](https://nifi.apache.org/download.html)
  * [NiFi Registry 0.1.0](https://nifi.apache.org/registry.html)
  * [Solr 7.2.1](https://lucene.apache.org/solr/downloads.html)

* Extract all the archives in a directory, and create two copies of NiFi

      nifi-1.5.0-1
      nifi-1.5.0-2
      nifi-registry-0.1.0
      solr-7.2.1

* Edit *nifi-1.5.0-2/conf/nifi.properties* and set a different web port

      nifi.web.http.port=7080

* Create directories for NiFi to ingest files from

      mkdir nifi-1.5.0-1/data
      mkdir nifi-1.5.0-2/data

* Download [this modified solrconfig.xml](https://gist.githubusercontent.com/anonymous/6c568bb06bd6393fda7fece360d344a5/raw/5d06e968193625c02e3a71e5423993a91efb721f/solrconfig.xml) and copy it Solr's default conf

      wget -O solrconfig.xml http://bit.ly/2EQrEbC
      cp solrconfig.xml solr-7.2.1/server/solr/configsets/\_default/conf/

* Start everything

      ./solr-7.2.1/bin/solr start -c
      ./nifi-registry-0.1.0/bin/nifi-registry.sh start
      ./nifi-1.5.0-1/bin/nifi.sh start
      ./nifi-1.5.0-2/bin/nifi.sh start

* Create Solr collections for DEV and PROD

      ./solr-7.2.1/bin/solr create -c data-dev -n \_default
      ./solr-7.2.1/bin/solr create -c data-prod -n \_default

* Verify Solr collections created successfully

  <img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/01-solr-collections.png" class="img-responsive img-thumbnail">

* At this point all the applications should be up and running at the following locations:

  * [http://localhost:8983/solr](http://localhost:8983/solr)
  * [http://localhost:18080/nifi-registry](http://localhost:18080/nifi-registry)
  * [http://localhost:8080/nifi/](http://localhost:8080/nifi/) (Dev NiFi)
  * [http://localhost:7080/nifi/](http://localhost:7080/nifi/) (Prod NiFi)

### Development Flow
