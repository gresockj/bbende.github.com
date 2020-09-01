---
layout: post
title: "Apache NiFi Schema Inference"
description: ""
category: "Development"
tags: [NiFi]
---
{% include JB/setup %}

In order to leverage NiFi's record processing capabilities, each record reader and record writer needs to obtain a schema
for the data. Schemas are defined using Apache Avro's schema representation which is a JSON document, and readers and
writers obtain schemas based on their configured "Schema Access Strategy".

The available strategies can vary based on the implementation of the reader/writer. For example, a CsvReader has an
option to create a schema of string fields using the header line to obtain the fields names, and an AvroReader has
an option to use an embedded schema if one exists. Some of the standard strategies that all readers and writers have are
to obtain a schema by name or id from a schema registry, obtain the schema text directly from a property value or flow file attribute, and for a writer to inherit the schema from the reader.

These options are quite flexible and solve many use-cases, but there are also many scenarios where defining a schema
is challenging. For example, maybe the incoming data is JSON with over a hundred fields and not all fields are
known ahead of time. 
