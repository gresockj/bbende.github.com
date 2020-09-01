---
layout: post
title: "Indexing Custom JSON with SolrJ"
description: ""
category: "Development"
tags: [Solr]
---
{% include JB/setup %}

During a recent Lucidworks webinar on the upcoming Apache Solr 5 release, Grant Ingersoll mentioned Solr's ability 
to index custom JSON documents. I was interested in using this feature and found this blog post demonstrating 
how to use the update handler - [Indexing Custom Json Data](http://lucidworks.com/blog/indexing-custom-json-data/).

My next thought was "how can I use this through SolrJ?" specifically, "how can I use this through CloudSolrServer to 
index JSON documents in a SolrCloud cluster?"  

In my previous experience, I always added documents to Solr by creating instances of SolrInputDocument and calling 
the add method on SolrServer, but In this case we would need to provide a JSON document as the body of the update.
So I set out to research how this could be accomplished through SolrJ.

#### SolrServer

First, we need to understand a little bit about what happens when you add a SolrInputDocument through SolrServer. The 
code below shows the add method from SolrServer:

    public UpdateResponse add(SolrInputDocument doc, int commitWithinMs) throws SolrServerException, IOException {
        UpdateRequest req = new UpdateRequest();
        req.add(doc);
        req.setCommitWithin(commitWithinMs);
        return req.process(this);
    }
    
The code is fairly simple and shows that UpdateRequest does most of the work we are interested in.

#### UpdateRequest

From looking at UpdateRequest I noticed a few things:

* It extends AbstractUpdateRequest which extends SolrRequest
* writeXML() produces the XML representing the documents to add
* getContentStreams() returns the XML from writeXML() as a List<ContentStream> 

At this point I started looking at ContentStream, and looked to see what classes extend AbstractUpdateRequest.

#### ContentStreamUpdateRequest

ContentStreamUpdateRequest was exactly the class I was looking for. This class allows you to specify a URL in the 
constructor (i.e. the URL of the update handler), and allows you to add files or ContentStreams that will be posted 
to the URL.

A ContentStream is just an interface providing methods such as getInputStream() and getReader(). SolrJ has several 
ContentStream implementations in ContentStreamBase, including:

* ByteArrayStream
* URLStream
* FileStream
* StringStream

Using this information we can construct a ContentStreamUpdateRequest for a custom JSON document like this:

    String myJson = ...
    
    ContentStreamUpdateRequest request = new ContentStreamUpdateRequest(
                "/update/json/docs");
    request.setParam("json.command", "false");
    request.setParam("split", "/exams");
    request.getParams().add("f", "first:/first");
    request.getParams().add("f", "last:/last");
    request.getParams().add("f", "grade:/grade");
    request.getParams().add("f", "subject:/exams/subject");
    request.getParams().add("f", "test:/exams/test");
    request.getParams().add("f", "marks:/exams/marks");

    request.addContentStream(new ContentStreamBase.StringStream(myJson));
    
    UpdateResponse response = request.process(solrServer);
    
#### Custom Update Request

One of the downsides to the above example was that the ContentStream required the whole JSON string to be in memory, 
and also required the user of the API to know a lot about the individual URL parameters. 

I thought it might be nice to stream a JSON document and avoid reading all of the content into memory. We can do this 
by extending AbstractUpdateRequest to implement our own custom update request, and at the same time provide an API 
specific to the JSON update handler:

    public class JSONUpdateRequest extends AbstractUpdateRequest {
        private final InputStream jsonInputStream;
    
        public JSONUpdateRequest(InputStream jsonInputStream) {
            super(METHOD.POST, "/update/json/docs");
            this.jsonInputStream = jsonInputStream;
            this.setParam("json.command", "false");
        }
    
        public void addFieldMapping(String field, String jsonPath) {
            getParams().add("f", field + ":" + jsonPath);
        }
    
        public void setSplit(String jsonPath) {
            setParam("split", jsonPath);
        }
    
        @Override
        public Collection<ContentStream> getContentStreams() throws IOException {
            ContentStream jsonContentStream = new InputStreamContentStream(
                    jsonInputStream, "application/json");
            return Collections.singletonList(jsonContentStream);
        }
    
        private class InputStreamContentStream extends ContentStreamBase {
            private final InputStream inputStream;
    
            public InputStreamContentStream(InputStream inputStream, 
              String contentType) {
                this.inputStream = inputStream;
                this.setContentType(contentType);
            }
    
            @Override
            public InputStream getStream() throws IOException {
                return inputStream;
            }
        }
    }

Using this to make the request looks similar to using the ContentStreamUpdateRequest:

    InputStream jsonInput = ...

    JSONUpdateRequest request = new JSONUpdateRequest(jsonInput);
    request.setSplit("/exams");
    request.addFieldMapping("first", "/first");
    request.addFieldMapping("last", "/last");
    request.addFieldMapping("grade", "/grade");
    request.addFieldMapping("subject", "/exams/subject");
    request.addFieldMapping("test", "/exams/test");
    request.addFieldMapping("marks", "/exams/marks");
    
    UpdateResponse response = request.process(solrServer);

#### Summary

Indexing custom JSON documents in Solr is an awesome feature, and using ContentStreamUpdateRequest allows 
you post any content, and can be used in conjunction with CloudSolrServer to add documents to a SolrCloud cluster.

Working examples for all of the code provided in this post are in GitHub:  
[solrj-custom-json-update](https://github.com/bbende/solrj-custom-json-update).