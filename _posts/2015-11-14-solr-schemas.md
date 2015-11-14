---
layout: post
title: "Solr Schemas"
description: ""
category: "Development"
tags: [Solr,Schema]
---
{% include JB/setup %}

When starting a new project involving Solr, usually one of the first tasks is deciding
how to define your schema, or how to *not* define your schema. Solr has made it increasingly
easier to start indexing data without much work, but there are some important choices to consider
when thinking about your schema.

### Standard Schema

In the standard schema approach, fields are defined ahead of time in schema.xml. Modifications
to the schema require editing schema.xml and reloading the core, or collection, or restarting
Solr. The basic_configs that come with Solr 5.3.1 demonstrates this approach.

In *solr-5.3.1/server/solr/configsets/basic_configs/conf/solrconfig.xml* we see:

    <schemaFactory class="ClassicIndexSchemaFactory"/>

This tells Solr to construct the classic schema which can not be modified while Solr
is running.

### Managed Schema with Field Guessing

At some point in time Solr introduced 'managed schema' which can be modified while Solr
is running using the schema REST API. The data_driven_schema_configs that come with
Solr 5.3.1 demonstrates this approach.

In *solr-5.3.1/server/solr/configsets/data_driven_schema_configs/conf/solrconfig.xml* we see:

    <schemaFactory class="ManagedIndexSchemaFactory">
        <bool name="mutable">true</bool>
        <str name="managedSchemaResourceName">managed-schema</str>
    </schemaFactory>

The documentation provided here states:

- *When ManagedIndexSchemaFactory is specified instead, Solr will load the schema from the resource named in 'managedSchemaResourceName', rather than from schema.xml.
Note that the managed schema resource CANNOT be named schema.xml.  If the managed
schema does not exist, Solr will create it after reading schema.xml, then rename
'schema.xml' to 'schema.xml.bak'.*

- *Do NOT hand edit the managed schema - external modifications will be ignored and
overwritten as a result of schema modification REST API calls.*

- *When ManagedIndexSchemaFactory is specified with mutable = true, schema
modification REST API calls will be allowed; otherwise, error responses will be
sent back for these requests.*

In addition to a managed schema, the data_driven_schema_configs also defines an update
chain that adds unknown fields to the schema based on incoming data. This is what most people
think of when they say "schema-less mode".

The update chain in this example looks like the following:

    <updateRequestProcessorChain name="add-unknown-fields-to-the-schema">
        <!-- UUIDUpdateProcessorFactory will generate an id if none is present in the incoming document -->
        <processor class="solr.UUIDUpdateProcessorFactory" />

        <processor class="solr.LogUpdateProcessorFactory"/>
        <processor class="solr.DistributedUpdateProcessorFactory"/>
        <processor class="solr.RemoveBlankFieldUpdateProcessorFactory"/>
        <processor class="solr.FieldNameMutatingUpdateProcessorFactory">
          <str name="pattern">[^\w-\.]</str>
          <str name="replacement">_</str>
        </processor>
        <processor class="solr.ParseBooleanFieldUpdateProcessorFactory"/>
        <processor class="solr.ParseLongFieldUpdateProcessorFactory"/>
        <processor class="solr.ParseDoubleFieldUpdateProcessorFactory"/>
        <processor class="solr.ParseDateFieldUpdateProcessorFactory">
          <arr name="format">
            <str>yyyy-MM-dd'T'HH:mm:ss.SSSZ</str>
            <str>yyyy-MM-dd'T'HH:mm:ss,SSSZ</str>
            <str>yyyy-MM-dd'T'HH:mm:ss.SSS</str>
            <str>yyyy-MM-dd'T'HH:mm:ss,SSS</str>
            <str>yyyy-MM-dd'T'HH:mm:ssZ</str>
            <str>yyyy-MM-dd'T'HH:mm:ss</str>
            <str>yyyy-MM-dd'T'HH:mmZ</str>
            <str>yyyy-MM-dd'T'HH:mm</str>
            <str>yyyy-MM-dd HH:mm:ss.SSSZ</str>
            <str>yyyy-MM-dd HH:mm:ss,SSSZ</str>
            <str>yyyy-MM-dd HH:mm:ss.SSS</str>
            <str>yyyy-MM-dd HH:mm:ss,SSS</str>
            <str>yyyy-MM-dd HH:mm:ssZ</str>
            <str>yyyy-MM-dd HH:mm:ss</str>
            <str>yyyy-MM-dd HH:mmZ</str>
            <str>yyyy-MM-dd HH:mm</str>
            <str>yyyy-MM-dd</str>
          </arr>
        </processor>
        <processor class="solr.AddSchemaFieldsUpdateProcessorFactory">
          <str name="defaultFieldType">strings</str>
          <lst name="typeMapping">
            <str name="valueClass">java.lang.Boolean</str>
            <str name="fieldType">booleans</str>
          </lst>
          <lst name="typeMapping">
            <str name="valueClass">java.util.Date</str>
            <str name="fieldType">tdates</str>
          </lst>
          <lst name="typeMapping">
            <str name="valueClass">java.lang.Long</str>
            <str name="valueClass">java.lang.Integer</str>
            <str name="fieldType">tlongs</str>
          </lst>
          <lst name="typeMapping">
            <str name="valueClass">java.lang.Number</str>
            <str name="fieldType">tdoubles</str>
          </lst>
        </processor>
        <processor class="solr.RunUpdateProcessorFactory"/>
      </updateRequestProcessorChain>

This update chain is then added to all update handlers with the following:

    <initParams path="/update/**">
        <lst name="defaults">
            <str name="update.chain">add-unknown-fields-to-the-schema</str>
        </lst>
    </initParams>

Walking through the update chain definition, we can see the following...

* If no id field is present on the incoming document, a UUID will be assigned by
UUIDUpdateProcessorFactory
* The incoming value is parsed into a Boolean, Long, Double, or Date, in that order,
to attempt determining the class of the value.
* AddSchemaFieldsUpdateProcessorFactory then maps the value class to a Solr field type,
using 'strings' as the default type for anything that could not be parsed into one
of the above classes.

As an example, if a document came in with a field named 'created_at' and a value of
'2015-11-14', ParseDateFieldUpdateProcessorFactory would successfully parse this into a
java.util.Date, and AddSchemaFieldsUpdateProcessorFactory would add a field named
'created_at' with type 'tdates', a multi-valued date type.

If a document came in with a field named 'first_name' and a value of 'bob', all of the
parsing attempts would fail, and AddSchemaFieldsUpdateProcessorFactory would add a new
field named 'first_name' with a type of 'strings', a multi-valued string type.

### Schemaless Gotchas

At this point you are probably thinking "why would anyone not use the schemaless approach
shown above?". Well there are a few things to consider when taking this approach.

#### Single-valued vs. Multi-valued

If you truly don't know your data well enough to define a schema, then it is hard to
know if an incoming field will be single-valued or multi-valued. For this reason, the
above example adds all fields as multi-valued to ensure documents won't be rejected.
The downside is that Solr can't sort on multi-valued fields, which is usually a desired
feature on date fields.  

You can obviously work around this by changing all the field mappings above to the
single-valued types (strings to string, booleans to boolean, etc.), but this only works
if you know all your fields will always be single-valued.

#### Guessing Wrong

Another possible scenario is that the first document that came in might have tricked Solr
into guessing the wrong type. Lets say the first document came in with a field named
'username' and a value of '123456789'. If you remember our update chain, the second type
it attempts to parse into is a Long which would be successful here. So Solr would add the
'username' field as type 'tlongs', and any documents that came in later with a username
containing non-numeric characters would fail.

You could work around this by using the REST API to delete the 'username' field and
add it back as a string type, but imagine having to do this several times.

### Dynamic Fields

Another option when deciding how to setup your schema is the use of dynamic fields.
Dynamic fields let Solr determine the field type from a pattern in the field name,
and can be used in both the standard and managed schema approaches. For example:

    <dynamicField name="*_i"  type="int"    indexed="true"  stored="true"/>

This tells Solr that any field name that ends with '_i' should be type 'int', and
should be indexed and stored.

A downside to dynamic fields is that you need control of the field naming on
incoming Solr documents so that they will match the dynamic field patterns. This
might not be possible in many situations.

If you are dealing with something like JSON, and don't have ability to control the
field names, you could also provide field mappings to Solr when the document is added.
However, if you need to map every field to a new name that matches a dynamic field name,
why not just define all these fields in your schema ahead of time?

### Summary

In summary, the answer to *"how should I define my Solr schema?"* is *"it depends..."*. Many
times a schema-less approach can be a great way to jump-start development and prototyping,
but creating a well defined schema later on can be beneficial as well. Using dynamic fields
also provides additional flexibility and can sometimes be a middle ground.
