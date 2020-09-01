---
layout: post
title: "Ambari Metrics Part 1 - Metrics Collector"
description: ""
category: "Development"
tags: [Ambari]
---
{% include JB/setup %}

Apache Ambari is a popular tool for provisioning, managing, and monitoring
Hadoop clusters. Part of Ambari is a metrics system for collecting, aggregating,
and serving metrics. For a complete description, check out the
[Ambari Metrics Wiki](https://cwiki.apache.org/confluence/display/AMBARI/Metrics).

This post will cover the steps to send your own metrics to the Ambari Metrics
Service, how to inspect the data using Phoenix, and how to interact with the
Metrics Service REST API.

### Initial Setup

One of the easiest ways to start using Ambari is by running the latest
[HDP Sandbox](http://hortonworks.com/hdp/downloads/) (version 2.3 at the time of
writing this). This post will assume you are able to run the sandbox, access the
Ambari web interface, and ssh to the virtual machine.

### Configure Port Forwarding

By default the port for the Metrics Collector service is not exposed
outside of the HDP Sandbox. In order to post metrics from your local machine,
add a port forwarding rule to the virtual machine:

* Right-Click the HDP 2.3_1 VM in VirtualBox and select Settings
* Select the Network panel, then Click Advanced, and then click Port Forwarding
* Add a new rule that maps Host Port 6188 to Guest Port 6188
* Restart the virtual machine if it was already running

### Start the Metrics Collector

When the HDP sandbox starts, the Metrics Collector service will not be running.

* Launch the Ambari web interface by going to localhost:8080 in a browser (admin/admin)
* Click on Ambari Metrics on the left hand menu
* Click Metrics Collector across the top
* Select the Start option for the Metrics Collector Service
* Verify that the Metrics Collector now says Started

<img src="{{ BASE_PATH }}/assets/images/ambari-metrics/metrics-started.jpg" class="img-responsive">

### Sending Metrics

Metrics are sent to the Metrics Collector through a [REST API](https://cwiki.apache.org/confluence/display/AMBARI/Metrics+Collector+API+Specification).
Using your favorite tool of choice (REST Client, Poster, etc.),  POST the example
JSON using the browser on your local machine:

    {
        "metrics": [
          {
              "metricname": "AMBARI_METRICS.SmokeTest.FakeMetric",
              "appid": "amssmoketestfake",
              "hostname": "ambari20-5.c.pramod-thangali.internal",
              "timestamp": 1432075898000,
              "starttime": 1432075898000,
              "metrics": {
                "1432075898000": 0.963781711428,
                "1432075899000": 1432075898000
              }
            }
            ]
    }

This should return a successful response with a status code of 200. If it is not
successful for any reason, inspect the log on the sandbox vm at
 /var/log/ambari-metrics-collector/ambari-metrics-collector.log.

### Phoenix

Behind the scenes the Ambari Metrics Service is using Apache Phoenix. The Phoenix
client can be used to verify the data that was sent. SSH to the Sandbox VM and download Phoenix 4.2.x:

    ssh root@127.0.0.1 -p 2222
    wget http://mirror.nexcess.net/apache/phoenix/phoenix-4.2.2/bin/phoenix-4.2.2-bin.tar.gz
    tar xzvf phoenix-4.2.2-bin.tar.gz
    cd phoenix-4.2.2-bin/bin/

In the bin directory, edit *sqlline.py* and replace any references to the Java
executable with the full path to Java, example: */usr/lib/jvm/java-1.7.0-openjdk.x86_64/bin/java*.
Connect to Phoenix and query for the test metric that was sent:

    ./sqlline.py localhost:61181:/hbase

    SELECT METRIC_NAME, SERVER_TIME, APP_ID
      FROM METRIC_RECORD WHERE APP_ID = 'amssmoketestfake'
      ORDER BY SERVER_TIME LIMIT 10;

This should return the one metric that was previously posted:

<img src="{{ BASE_PATH }}/assets/images/ambari-metrics/metrics-phoenix.jpg" class="img-responsive">

To see all of the columns available on the METRIC_RECORD table, type:

    !describe METRIC_RECORD

### Retrieving Metrics

Using the results from Phoenix, we can now verify that the retrieval side of the
REST API works. Navigate to:

    http://localhost:6188/ws/v1/timeline/metrics?metricNames=AMBARI_METRICS.SmokeTest.FakeMetric&appId=amssmoketestfake&hostname=ambari20-5.c.pramod-thangali.internal&precision=seconds&startTime=1438365056401&endTime=1438365056403

The key here is that the startTime and endTime are respectively before and after the
SERVER_TIME value in the Phoenix results. This should return a JSON result that matches
up with the metric that was posted earlier.

**NOTE**: Currently the retrieval side will always query for the appId parameter as a lowercase value,
so make sure you are sending metrics with a lowercase appId.

### Summary

This post showed how to send metrics to Ambari and interact with the Metrics
Service REST API, as well as the Phoenix client. Part 2 will demonstrate how to add
a service to Ambari with widget
