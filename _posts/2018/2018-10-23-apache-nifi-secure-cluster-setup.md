---
layout: post
title: "Apache NiFi Secure Cluster Setup"
description: ""
category: "Development"
tags: [NiFi]
---
{% include JB/setup %}

Setting up a secure cluster continues to generate a lot of questions, so even though several posts have already covered
this topic, I thought I'd document the steps I performed while verifying the Apache NiFi 1.8.0 Release Candidate.

### Initial Setup

The simplest cluster you can setup for local testing is a two node cluster, with embedded ZooKeeper running on the first node.

After building the source for 1.8.0-RC3, unzip the binary distribution and create two identical directories:

    unzip nifi-1.8.0-bin.zip
    mv nifi-1.8.0 nifi-1
    cp -R nifi-1 nifi-2

The resulting directory structure should look like the following:

    cluster
    ├── nifi-1
    │   ├── LICENSE
    │   ├── NOTICE
    │   ├── README
    │   ├── bin
    │   ├── conf
    │   │   ├── authorizers.xml
    │   │   ├── bootstrap-notification-services.xml
    │   │   ├── bootstrap.conf
    │   │   ├── logback.xml
    │   │   ├── login-identity-providers.xml
    │   │   ├── nifi.properties
    │   │   ├── state-management.xml
    │   │   └── zookeeper.properties
    │   ├── docs
    │   └── lib
    └── nifi-2
        ├── LICENSE
        ├── NOTICE
        ├── README
        ├── bin
        ├── conf
        │   ├── authorizers.xml
        │   ├── bootstrap-notification-services.xml
        │   ├── bootstrap.conf
        │   ├── logback.xml
        │   ├── login-identity-providers.xml
        │   ├── nifi.properties
        │   ├── state-management.xml
        │   └── zookeeper.properties
        ├── docs
        └── lib

### Configure nifi.properties

Edit *nifi-1/conf/nifi.properties* and set the following properties:

    nifi.state.management.embedded.zookeeper.start=true

    nifi.remote.input.secure=true

    nifi.web.http.port=
    nifi.web.https.host=localhost
    nifi.web.https.port=8443

    nifi.security.keystore=/path/to/keystore.jks
    nifi.security.keystoreType=jks
    nifi.security.keystorePasswd=yourpassword
    nifi.security.keyPasswd=yourpassword
    nifi.security.truststore=/path/to/truststore.jks
    nifi.security.truststoreType=jks
    nifi.security.truststorePasswd=yourpassword

    nifi.cluster.protocol.is.secure=true

    nifi.cluster.is.node=true
    nifi.cluster.node.protocol.port=8088
    nifi.cluster.flow.election.max.candidates=2

    nifi.zookeeper.connect.string=localhost:2181

Edit *nifi-2/conf/nifi.properties* and set the following properties:

    nifi.remote.input.secure=true

    nifi.web.http.port=
    nifi.web.https.host=localhost
    nifi.web.https.port=7443

    nifi.security.keystore=/path/to/keystore.jks
    nifi.security.keystoreType=jks
    nifi.security.keystorePasswd=yourpassword
    nifi.security.keyPasswd=yourpassword
    nifi.security.truststore=/path/to/truststore.jks
    nifi.security.truststoreType=jks
    nifi.security.truststorePasswd=yourpassword

    nifi.cluster.protocol.is.secure=true

    nifi.cluster.is.node=true
    nifi.cluster.node.protocol.port=7088
    nifi.cluster.flow.election.max.candidates=2
    nifi.cluster.load.balance.port=6343

    nifi.zookeeper.connect.string=localhost:2181

NOTE: For nifi-1 I left the default value for nifi.cluster.load.balance.port and since we are running both nodes on the
same host, we need to set a different value for nifi-2.

### Configure authorizers.xml

The configuration of authorizers.xml should be the same for both nodes, so edit *nifi-1/conf/authorizers.xml* and
*nifi-2/conf/authorizers.xml* and do the following...

Modify the userGroupProvider to declare the initial users for the initial admin and for the cluster nodes:

    <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Initial User Identity 1">CN=bbende, OU=ApacheNiFi</property>
        <property name="Initial User Identity 2">CN=localhost, OU=NIFI</property>
    </userGroupProvider>

NOTE: In this case since both nodes are running on the same host and using the same keystore, there only needs to be
one user representing both nodes, but in a real setup there would be multiple node identities.

NOTE: The user identities are case-sensitive and white-space sensitive, so make sure the identities are entered exactly
as they will come across to NiFi when making a request.

Modify the accessPolicyProvider to declare which user is the initial admin and which are the node identities:

    <accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <property name="Initial Admin Identity">CN=bbende, OU=ApacheNiFi</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Node Identity 1">CN=localhost, OU=NIFI</property>
        <property name="Node Group"></property>
    </accessPolicyProvider>

### Configure state-management.xml

The configuration of state-management.xml should be the same for both nodes, so edit *nifi-1/conf/state-management.xml* and
*nifi-2/conf/state-management.xml* and do the following...

Modify the cluster state provider to specify the ZooKeeper connect string:

    <cluster-provider>
        <id>zk-provider</id>
        <class>org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider</class>
        <property name="Connect String">localhost:2181</property>
        <property name="Root Node">/nifi</property>
        <property name="Session Timeout">10 seconds</property>
        <property name="Access Control">Open</property>
    </cluster-provider>

### Configure zookeeper.properties

The configuration of zookeeper.properties should be the same for both nodes, so edit *nifi-1/conf/zookeeper.properties* and
*nifi-2/conf/zookeeper.properties* and do the following...

Specify the servers that are part of the ZooKeeper ensamble:

    server.1=localhost:2888:3888

### Create ZooKeeper myid file

Since embedded ZooKeeper will only be started on the first node, we only need to do the following for nifi-1:

    mkdir nifi-1/state
    mkdir nifi-1/state/zookeeper
    echo 1 > nifi-1/state/zookeeper/myid

### Start Cluster

    ./nifi-1/bin/nifi.sh start && ./nifi-2/bin/nifi.sh start

At this point, assuming you have the client cert (p12) for the initial admin in your browser ("CN=bbende, OU=ApacheNiFi")
then you can access your cluster at https://localhost:8443/nifi or https://localhost:7443/nifi.

### Common Issues

1. *Embedded ZooKeeper fails to create directory*

        java.io.IOException: Unable to create data directory ./state/zookeeper/version-2
            at org.apache.zookeeper.server.persistence.FileTxnSnapLog.<init>(FileTxnSnapLog.java:85)
            at org.apache.nifi.controller.state.server.ZooKeeperStateServer.startStandalone(ZooKeeperStateServer.java:85)
            ... 51 common frames omitted

      This happens frequently with embedded ZooKeeper and usually you can just try to start again and it will resolve itself.

2. *Mistake in initial admin identity, or node identity*

      If you made a typo in any of the identities and need to edit authorizers.xml, you MUST delete conf/users.xml and
      conf/authorizations.xml from each node before restarting.

      The reason is that the initial admin and node identities are only used when NiFi starts with no other users, groups,
      or policies. If you don't delete the users and authorization files, then your changes will be ignored.
