---
layout: post
title: "Indexing Wikipedia With Apache Solr"
description: ""
category: "Development"
tags: [Solr, Java]
---
{% include JB/setup %}

Wikipedia dumps are one of the best freely available, text-based data sets for research. I recently 
wrote some Java code to parse the bzip dump files and index the data with Apache Solr. The code is 
available on GitHub [here](https://github.com/bbende/solr-wikipedia) and should be farily easy to
extend from for other purposes. The code can be used as a stand-alone parser, and can even be 
customized to parse the data into a different object model than the one provided.

After getting the code working I ran a few tests to see what kind of performance I could get when 
ingesting an entire dump file into a single Solr instance on my laptop. The following is the 
configuration of my system when running these tests:

> #### Hardware
>
>  * MacBook Pro
>  * 2.7 GHz Core I7
>  * 16GB Ram
>  * 500GB SSD
>
> #### Solr
>
>  * Version 4.9
>  * autoSoftCommit = 10s
>  * autoHardCommit = 60s
>  * termVectors, termPositions, termOffests = true for text and title fields
>  * Jdk 7u67, 4GB heap, default garbage collection
> 
> #### Indexer
>
>  * StAX based XML parsing using Apache Commons bzip2 InputStream
>  * Configurable batching of documents (defaults to 20)
>  * Jdk 7u67, 1GB heap, default garbage collection
> 
> #### Data
>
>  * The english Wikipedia latest articles data set
>  * 11 GB compressed, 14 million documents
 

Using a batch size of 40 documents and a single-threaded ingest process, the entire dump
file was indexed in just over three hours. The resulting index is 51 GB on disk
and contains 14,650,372 documents. I tested a couple of variations of the Indexer using
multiple threads, but did not observe any noticeable increase in performance. I suspect 
this is because the parsing can't be parallelized since we are only dealing with a single 
file as an input stream, and adding a batch of documents to Solr seemed to be generally 
low-latency with no network communication and an SSD. 

In the future I would be interested in testing out the ConucrrentUpdateSolrServer to see
if this produces any performance gains, as opposed to my own attempt at making the indexer
multi-threaded. It would also be interesting to index the data with a shareded SolrCloud 
where multiple nodes could be receiving updates and indexing in parallel. If anyone does
anything interesting with this code, or with the data after indexing it in Solr, let me 
know.