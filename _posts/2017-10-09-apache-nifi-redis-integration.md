---
layout: post
title: "Apache NiFi - Redis Integration"
description: ""
category: "Development"
tags: [NiFi, Redis]
---
{% include JB/setup %}

The 1.4.0 release of Apache NiFi contains a new Distributed Map Cache (DMC) Client that interacts with Redis as the
back-end cache implementation.

This post will give an overview of the traditional DMC, show an example of how to use the Redis DMC Client with existing processors, and discuss how Redis can be configured for high-availability.

### Distributed Map Cache

The Distributed Map Cache (DMC) Client is a controller service for interacting with a cache. NiFi has
traditionally provided an implementation of the DMC Client that talks to a DMC Server.

There are a number of processor that make use of a cache through the DMC Client interface. Some of these processors interact with the cache directly, such as PutDistributedMapCache and FetchDistributedMapCache, and others leverage
the cache behind the scenes, such as DetectDuplicate or Wait/Notify.

In a typical NiFi cluster, the DMC Server is started on all nodes, but the DMC Client points to only one of the servers,
which would look like this:

<img src="{{ BASE_PATH }}/assets/images/nifi-redis/01-traditional-dmc.png" class="img-responsive img-thumbnail">

A benefit of the DMC Server is that it can be fully managed by NiFi and doesn't rely on any external processes or
third-party libraries. However, a limitation is that it was not built for high-availability. All of the cache data is in the one DMC server being used by the clients, and if that node goes down, there is no fail-over to one of the other nodes.

The good news is that we can easily provide additional implementations of the DMC Client that leverage alternative
key/value stores. Since other components only know about the DMC Client interface, the implementation can be swapped
out without changing those components. This means processors such as PutDistributedMapCache and FetchDistributedMapCache will work exactly the same.

### Redis DMC Client

The 1.4.0 release of NiFi introduces the Redis DMC Client. In this case, there is no longer a DMC Server running in NiFi. The setup would look like the following:

<img src="{{ BASE_PATH }}/assets/images/nifi-redis/02-redis-dmc.png" class="img-responsive img-thumbnail">

The following example will demonstrated how to use the existing PutDistributedMapCache and FetchDistributedMapCache processors to interact with Redis.

* Start Redis

      ./redis-server redis.conf

* Create a RedisConnectionPoolService

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/03-redis-connection-pool.png" class="img-responsive img-thumbnail">

    Select "Redis Mode" of "Standalone" and enter the default "Connection String" of "localhost:6379".

* Create a RedisDistributedMapCacheClientService

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/04-redis-dmc-service.png" class="img-responsive img-thumbnail">

    Select the RedisConnectionPoolService created in the previous step.

* Create a GenerateFlowFile processor

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/05-generate-flow-file.png" class="img-responsive img-thumbnail">

    Set the "Custom Text" property to ${now()} which will put the current date & time into the content of each flow file,
    and change the Run Schedule to 1 second.

* Create a PutDistributedMapCache processor

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/06-put-dmc.png" class="img-responsive img-thumbnail">

    Select the Redis DMC Client Service and set the "Cache Entry Identifier" to "date" which will be the key in Redis.

* Create a FetchDistributedMapCache processor

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/07-fetch-dmc.png" class="img-responsive img-thumbnail">

    Select the Redis DMC Client Service and use the same "Cache Entry Identifier" that was used in PutDistributedMapCache. Also
    set "Put Cache Value in Attribute" to put the fetched value into an attribute named "date.retrieved".

* Create a LogAttribute processor, connect everything, and run the flow

    <img src="{{ BASE_PATH }}/assets/images/nifi-redis/08-full-flow.png" class="img-responsive img-thumbnail">

* Checking nifi-app.log, each flow file passing through LogAttribute should show "data.retrieved"

        FlowFile Attribute Map Content
        Key: 'cached'
        	Value: 'true'
        Key: 'date.retrieved'
        	Value: 'Mon Oct 02 10:22:08 EDT 2017'

If you completed the above example, you should have a working flow that is storing the current date in Redis under a key
name "date", and then retrieving that value into a flow file attribute name "date.retrieved".

### Redis Sentinel

The above example used a standalone Redis which still leaves us with a single point of failure. We can
improve upon this by using [Redis Sentinel](https://redis.io/topics/sentinel) which provides high-availability for Redis.

I'll assume you can follow the [Quick Tutorial](https://redis.io/topics/sentinel#a-quick-tutorial) for setting
up Redis Sentinel.

This example assumes we have three sentinel processes running on ports 5000, 5001, and 5002, and two Redis
instances running on ports 6379 and 6380, where 6380 is initially a replica of 6379.

We can then re-configure the RedisConnectionPoolService...

<img src="{{ BASE_PATH }}/assets/images/nifi-redis/09-redis-sentinel.png" class="img-responsive img-thumbnail">

We change the "Redis Mode" to "Sentinel" and the "Connection String" becomes the comma-separated list of sentinels. We also
specified the "Sentinel Master" name as the default name of "mymaster".

At this point we can start our flow again and everything should be working as it was before.

If we then kill the Redis server running at 6379, we should see a fail-over take place in the sentinel logs:

    28792:X 02 Oct 10:58:45.055 # +sdown master mymaster 127.0.0.1 6379

    28792:X 02 Oct 10:58:45.107 # +new-epoch 10
    28792:X 02 Oct 10:58:45.107 # +vote-for-leader 68203d8ca572b2d79c939124767ba1746623a362 10
    28792:X 02 Oct 10:58:45.110 # +odown master mymaster 127.0.0.1 6379 #quorum 3/2
    28792:X 02 Oct 10:58:45.110 # Next failover delay: I will not start a failover before Mon Oct  2 11:00:45 2017

    28792:X 02 Oct 10:58:46.188 # +config-update-from sentinel 68203d8ca572b2d79c939124767ba1746623a362 127.0.0.1 5002 @ mymaster 127.0.0.1 6379
    28792:X 02 Oct 10:58:46.188 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6380

We can see that the sentinels detected when the server on port 6379 went down, and and then changed the server on port 6380
to be the new master (last line of the logs).

At the same time, the PutDistributedMapCache processor would show an error about being unable to retrieve a connection
from the connection pool. This happens while the sentinels are in process of performing the fail-over.

While this is happening, flow files would begin to queue up until the fail-over was completed. Once the new master is elected,
all the flow files would then be processed.

### Redis Cluster

In Redis Sentinel, each Redis server has a copy of the entire data set. In some cases it may be desirable to shard the
key space across several Redis servers, where each server only has a portion of the data set.

Unfortunately, the Redis DMC Client Service must be able to perform atomic compare-and-set operations which
are implemented with Redis transactions, and Redis transactions are not supported in a cluster.

So for the Redis DMC Client Service we are limited to standalone Redis, or Redis Sentinel.
