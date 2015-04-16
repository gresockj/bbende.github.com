---
layout: post
title: "Concatenating Fields with Apache Solr"
description: ""
category: "Development"
tags: [Solr]
---
{% include JB/setup %}

Apache Solr's ConcatFieldUpdateProcessorFactory allows you to concatenate the values of a field when adding a 
document to the index. Using this feature, in conjunction with the CloneFieldUpdateProcessorFactory, allows you 
to create a new field that is the concatenation of other fields in the document. 

This can be particularly useful 
when working with CSV data where you might be using the CSV update handler, and don't want to write code to 
pre-process the data.

As an example, lets say we have a CSV file with three values on each row. When we add these records to 
Solr we want to create a fourth value that is the combination of the other three values, delimited by '_'s. 
If the first line was 'a, b, c' this would produce 'a, b, c, a_b_c'.

The first step is to add a new updateRequestProcessorChain to solrconfig.xml:

    <updateRequestProcessorChain name="concatFields">
    	<processor class="solr.CloneFieldUpdateProcessorFactory"> 
	    	<str name="source">field1</str> 
	    	<str name="dest">field4</str> 
	    </processor> 
    	<processor class="solr.CloneFieldUpdateProcessorFactory"> 
	    	<str name="source">field2</str> 
	    	<str name="dest">field4</str> 
	    </processor> 
	    <processor class="solr.CloneFieldUpdateProcessorFactory"> 
	    	<str name="source">field3</str> 
	    	<str name="dest">field4</str> 
	    </processor> 
	    <processor class="solr.ConcatFieldUpdateProcessorFactory"> 
		    <str name="fieldName">field4</str> 
	    	<str name="delimiter">_</str> 
	    </processor>
	    <processor class="solr.LogUpdateProcessorFactory" /> 
	    <processor class="solr.RunUpdateProcessorFactory" />
    </updateRequestProcessorChain> 
 
The update chain clones the other three fields to the fourth field, and the 
ConcatFieldUpdateProcessorFactory concatenates the values of the fourth field using the given delimiter.

After adding the update chain, the next step is to add that chain to the appropriate update handler, in this 
case the CSV handler.

    <requestHandler name="/update/csv" class="solr.UpdateRequestHandler">
         <lst name="defaults">
         <str name="stream.contentType">application/csv</str>
         <str name="update.chain">concatFields</str>
       </lst>
    </requestHandler>
    
After restarting Solr you should be able to index new documents and see the new field getting populated.