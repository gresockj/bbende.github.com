---
layout: post
title: "Lucene Solr Revolution 2014"
description: ""
category: "General"
tags: [Solr, Lucene]
---
{% include JB/setup %}

This past week I attended Lucene/Solr Revolution for the second year in a row. This year's
event was held in Washington D.C with great talks from companies such as Bloomberg, StubHub, 
Target, AirBnb, and Apple, just to name a few. Many of these talks showed how Solr has matured 
and evolved beyond just a standard search engine. A lot of companies are using Solr in unique 
ways at the center of their architecture.

Here are some highlights from the talks I attended:

#### Day 1

- Bloomberg took an interesting approach for creating different views of a document. They initially 
tried dynamic fields with a permission encoded at the end of the field name, but that didn't scale 
well. They eventually encoded the permission as a prefix of each token on the value side, for 
example text: "view1_val1 view2_val1 ...".

- MapR gave a presentation showing how machine learning can be integrated with Solr. The example 
architecture used Pig to pre-process logs, Mahout to generate recommendations, and Python to add the 
recommendations to the existing documents in Solr for easy retrieval/presentation. 

- Bloomberg built an analytic component from the ground up that is available as a patch for 
Solr 4.X through [SOLR-5302](https://issues.apache.org/jira/browse/SOLR-5302), and will be a 
contrib module for Solr 5.x.

- StubHub presented an interesting use case of using Solr to help de-duplicate events and
venues as they are added to the system. They originally used MySQL, but moved to a Solr
solution utilizing a geo-distance function, a custom update handler, and a scorer to determine
if an event or venue was similar enough to be considered the same.

- Evernote has over 100 million users with 1 Lucene index per user, and they migrated 
to Lucene 4.5 with an average downtime of less than 1 minute per user. They run multiple versions 
of Lucene per server by building the version into the package name and utilizing custom class 
loaders. They significantly decreased their disk I/O by migrating Lucene to 4.5, and made other 
performance improvements involving compression.

- Mikhail Khludnev of Grid Dynamics gave an interesting presentation on the different types of joins 
in Solr, along with a possible new approach. The existing joins are BlockJoin and JoinUtil. BlockJoin is 
fast at search time, but slow during updates, and JoinUtil is slow at search time, but fast during 
indexing. Mikhail's proposed approach created a JoinIndex of document ids using DocValues, 
but was dependent on updateable DocValues. Mikhail's slides are available [here](http://goo.gl/aYsxiD).  

#### Day 2

- Apple discussed a large SolrCloud deployment where they put approximately 1,000 logical indexes per collection with 
custom sharding and three-level id routing. They developed an automation tool called SolrLord which is capable 
of monitoring the environment and taking actions, such as adding replicas, migrating replicas, and splitting shards. 
They implemented cross data-center replication by using a wrapper for SolrJ which inserts documents to the primary 
cluster, and also queues documents for asynchronous insertion to a second data center.

- Shalin Shekhar Mangar presented some of the challenges and improvements that have been made scaling SolrCloud. He 
mentioned using the Solr Scale Toolkit to perform a stress test with 30 hosts, 120 nodes, 1k collections, 6B docs, 
15k queries/sec, 2k writes/sec, and 2 sec sustained NRT search.

- Amazon ported their AWS CloudSearch service from a proprietary engine to Solr. A custom ingest pipeline performs 
validation and writes data to S3 before inserting to Solr. The query side of the API hits Solr directly, but uses 
custom search components so the API is slightly different from Solr. Elastic Map Reduce is used to rebuild the index on 
configuration changes, or to split the index when scaling.

- Target incorporated Solr into their architecture and initially prototyped the type-ahead feature on the main 
Target search page. As part of moving to a micro-service architecture with continuous delivery they started using 
a collection alias. The alias points to the main catalog collection and they prepare a shadow catalog collection 
in the background with any necessary updates. At deployment time they swap the alias to have a collection that is 
immediately ready to use.

- LinkedIn built a custom search architecture called Galene which uses the Lucene index format on disk. A major 
requirement was "early termination" which allows them to stop returning results for a query after an acceptable 
 number of results have been returned. Implementing this feature requires the index to be sorted by rank. 

Looking forward to Lucene Solr Revolution 2015!