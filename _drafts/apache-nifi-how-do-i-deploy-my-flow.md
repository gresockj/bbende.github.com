---
layout: post
title: "Apache NiFi - How do I deploy my flow?"
description: ""
category: "Development"
tags: [NiFi, Solr]
---
{% include JB/setup %}

* TOC
{:toc}

### Introduction

One of the most frequently asked questions about NiFi has been *"How do I deploy my flow?"*.

A core feature of NiFi is that you can modify the live data flow without having to perform the traditional design and deploy steps. As a result, the idea of "deploying a flow" wasn't really baked into the system from the beginning.

Over time there were some attempts at solving this problem, such as putting the flow.xml.gz under version control, or scripting the deployment of templates, but each of these had it's own issues and difficulties.

In order to solve the problem, the community put a significant amount of effort into building direct support for version controlling data flows. The result of this effort was the creation of a whole new Apache NiFi sub-project, called *NiFi Registry*.

NiFi Registry is a complementary application that provides a central location for storage and management of shared resources. The first release focused on storing and managing versioned flows, and the 1.5.0 release of NiFi added the ability to interact with registry instances.

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

* Use NiFi to ingest this data to Solr, converting from CSV to JSON
* Add a full_name field with the combination of first name + last name
* Add a gender field based on the title
* Ingest to different Solr collections depending on environment   

To simulate this scenario we are going to setup two NiFi instances to represent DEV and PROD, a Solr instance with DEV and PROD collections, and NiFi Registry instance facilitate deploying our flow.

### Environment Setup

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

To get started with the development flow, download this template and import it the first NiFi instance:

* [Solr-Ingest-User-Data.xml](https://gist.githubusercontent.com/bbende/3ab0c7a0a13f33253435fed57b8c8798/raw/9ab0453a2444e971d35168e99f5a38a04f1925f9/Solr-Ingest-User-Data.xml)

After importing the template you should be able to instantiate it and end up with the following process group:

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/02-nifi-dev-pg.png" class="img-responsive img-thumbnail">

Going into this process group should show the following flow:

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/03-nifi-dev-flow.png" class="img-responsive img-thumbnail">

NOTE: You will have to enable all of the controller services and start all of the processors to get everything running after instantiating the template.

In order to process the data we are going to use some record-based processors which means we need a schema. The schema we are going to use is the following and can be seen in the AvroSchemaRegistry controller service:

    {
    "type": "record",
    "name": "user",
    "fields": [
      { "name": "title", "aliases" : [ "name.title" ], "type": "string" },
      { "name": "first_name", "aliases" : [ "name.first" ], "type": "string" },
      { "name": "last_name", "aliases" : [ "name.last" ], "type": "string" },
      { "name": "full_name", "type": ["string", "null"] },
      { "name": "gender", "type": ["string", "null"] },
      { "name": "email", "type": "string" },
      { "name": "registered", "type": "string" }
    ]
    }

Notice that gender and full_name are nullable since they are not part of our initial data and we are going to populate them later. We are also using aliases

We're not going to look at the configuration of every processor, but here is a brief description of what each processor is doing:

* *GetFile* is picking up files in ./data under the give NiFi's home directory
* *UpdateAttribute* adds a "schema.name" attribute with a value of "user"
* *ConvertRecord* uses a CSV reader and JSON writer to convert from CSV to JSON
* *UpdateRecord* populates the full_name field by concatenating first_name and last_name
* *LookupRecord* populates the gender field by passing the value of the title field to a lookup service
* *PutSolrContentStream* sends the data to Solr

For the Solr processor we have set the Collection property to *${solr.collection}* so that we can reference a variable. If we right-click on the canvas somewhere and select "Variables", we can see the definition of this variable.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/04-nifi-dev-vars.png" class="img-responsive img-thumbnail">

At this point we can ingest some data by copying the users-10.csv from earlier to the first NiFi's ingest directory:

    cp users-10.csv nifi-1.5.0-1/data

We can then verify this data was ingested to the *data-dev* collection in Solr:

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/05-solr-data-dev.png" class="img-responsive img-thumbnail">

### Starting Version Control

We now have everything working in our development environment and want to deploy this flow to production.

To get started we need to create a bucket in our NiFi Registry instance. Buckets are used as a way to organize items in the registry, and as mechanism to assign permissions to restrict access to data in the registry when running secured.

Clicking the tool icon in the top-right of the registry brings us to the bucket administration panel, and from there we can create a new bucket called "Solr Flows".

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/06-nifi-reg-bucket.png" class="img-responsive img-thumbnail">

Back in our development NiFi (port 8080), we need to tell NiFi about our registry instance. We can do that from Controller Settings under the menu in the top-right corner, by selecting the Registry Client tab and adding a new client.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/07-nifi-reg-clients.png" class="img-responsive img-thumbnail">

Once there is a registry client defined, there should now be a new "Version" menu available when right-clicking on process groups.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/08-nifi-dev-start-vc.png" class="img-responsive img-thumbnail">

Selecting "Start version control" will prompt us to select a registry instance, choose a bucket, and give our flow a name and description to save in the registry.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/09-nifi-dev-save-v1.png" class="img-responsive img-thumbnail">

Clicking Save will send the first version of this flow to our registry instance. We can also see that our process group is under version control and in-sync with the latest version in the registry.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/10-nifi-dev-under-vc.png" class="img-responsive img-thumbnail">

Refreshing the main page of the registry, we can see the version history for our flow.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/11-nifi-reg-items.png" class="img-responsive img-thumbnail">

### Deploying to Production

Now that we have our flow under version control and saved to the registry, we just need to import that flow into our
production NiFi.

From the second NiFi instance (port 7080), we need to define the registry client just like we did in the previous section. See the steps from the previous section.

After defining the registry client, we can drag a process group on to the canvas, and we'll have a new option to create a process group by importing a versioned flow.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/12-nifi-prod-add-pg.png" class="img-responsive img-thumbnail">

Selecting "Import" will prompt us to select the registry, bucket, flow, and version to import.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/13-nifi-prod-import.png" class="img-responsive img-thumbnail">

After selecting our flow, we now have the exact same flow in our production environment.

Remembering that we want to use a different Solr collection in production, we need to edit the variables so that *${solr.collection}* points to our production collection (data-prod).

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/14-nifi-prod-vars.png" class="img-responsive img-thumbnail">

We also have to enable all of the controller services and start all of the processors to get the flow running.

NOTE: These two operations of editing variables and enabling/starting components are only necessary on the first import to get everything setup. On future updates these values will be retained.

With everything running, we can now copy the users-10.csv file to the data directory under the second NiFi:

    cp users-10.csv nifi-1.5.0-2/data/

Checking the data-prod collection in Solr, we can now see the data was ingested to our production collection.

<img src="{{ BASE_PATH }}/assets/images/nifi-deploy-flow/15-solr-data-prod.png" class="img-responsive img-thumbnail">

### Deploying Updates
