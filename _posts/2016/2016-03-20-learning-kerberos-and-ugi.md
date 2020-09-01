---
layout: post
title: "Learning Kerberos and UGI"
description: ""
category: "Development"
tags: [Kerberos, UGI]
---
{% include JB/setup %}

Kerberos is the standard authentication mechanism in the Hadoop ecosystem. When a
service is secured with Kerberos, or as they say "Kerberized", client connections must
authenticate with a principal and keytab before being able to interact with the service.

The standard way to authenticate from Java code is through the UserGroupInformation (UGI)
class provided in the hadoop-common module. The most recent version of hadoop-common at
the time of writing this is 2.7.2 which can be used through the following Maven dependency:

    <dependency>
	    <groupId>org.apache.hadoop</groupId>
	    <artifactId>hadoop-common</artifactId>
	    <version>2.7.2</version>
    </dependency>

### Configuration

Before authenticating, the UGI class must be initialized with the Configuration
of the service by calling:

    Configuration conf = ...;
    UserGroupInformation.setConfiguration(conf);

The Configuration object is the standard Hadoop Configuration object. The JavaDocs for
Configuration say:

> Unless explicitly turned off, Hadoop by default specifies two resources, loaded in-order from the classpath:
>
> core-default.xml: Read-only defaults for hadoop. <br/>
> core-site.xml: Site-specific configuration for a given hadoop installation.
>
> Applications may add additional resources, which are loaded subsequent to these resources in the order they are added.

### isSecurityEnabled()

Once UGI has been initialized with the configuration, an application can call *isSecurityEnabled()*
to check if authentication is required. This method essentially checks the configuration to see if
a property with the name *"hadoop.security.authentication"* is set to *"kerberos"*.

One important note is that *"hadoop.security.authentication"* is typically set in core-site.xml,
so it is important to ensure the appropriate resources are loaded into the Configuration instance
being set in UGI. For example, if you initialized UGI with a Configuration instance that had only
hbase-site.xml loaded, then *isSecurityEnabled()* would likely return false, even though it may be
a kerberized HBase.

### Authenticating

After determining that security is enabled, the next step is to perform the login with a given
principal and keytab. There are two static methods on UGI to perform this operation:

* static void loginUserFromKeytab(String user, String keytab)
* static UserGroupInformation loginUserFromKeytabAndReturnUGI(String user, String keytab)

The first method performs the authentication and alters the static state in UGI so that
*getLoginUser()* will return the authenticated UGI instance. This approach is appropriate
if your application only authenticates as a single principal.

The second method performs the authentication and does not alter any static state in UGI,
and instead returns the authenticated UGI instance for the application to store. This is
more appropriate if your application authenticates against multiple services with different
principals.

### Privileged Actions

After authenticating, the UGI instance can now perform privileged actions. This
is done through the *doAs(PrivilegedAction action)* method on UGI.

In many cases it is only the first action that needs to be privileged, such as obtaining a connection, and then all operations on that connection will be performed as that user. As an example of creating a connection to HBase:

    Connection connection = ugi.doAs(new PrivilegedExceptionAction<Connection>() {
        @Override
        public Connection run() throws Exception {
            return ConnectionFactory.createConnection(hbaseConfig);
        }
    });

After obtaining the Connection using doAs, all the operations on the connection can
be performed as they normally would be.

### Multiple UGIs

As described above, if your application authenticates to multiple services with different
principals, then it is important to NOT rely on the static state of UGI, and instead  use
the *loginUserFromKeytabAndReturnUGI* method.

In addition, it is important to ensure that UGI is initialized with the appropriate
Configuration when performing the authentication. As an example, take the following
code:

    Configuration conf = ...;
    UserGroupInformation.setConfiguration(conf);

    UserGroupInformation ugi = null;
    if (UserGroupInformation.isSecurityEnabled()) {
      ugi = UserGroupInformation.loginUserFromKeytabAndReturnUGI(user, keytab);
    }

Imagine this code running concurrently in two different threads in the same JVM, each
with different Configuration instances, and each with different principals and keytabs.
The first thread could set the static configuration in UGI, but before checking
*isSecurityEnabled()* or performing the login, the second thread overwrites the static
configuration.

In order to prevent this, the application must implement some synchronization/locking strategy
to ensure that setting the configuration and performing the authentication happen atomicly.
