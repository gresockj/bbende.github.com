---
layout: post
title: "Apache NiFi 1.14.0 - Secure by Default"
description: ""
category: "Development"
tags: [NiFi]
---
{% include JB/setup %}

One of the major improvements in Apache NiFi 1.14.0 was to enable security for the default configuration. This means
all you have to do now is run `bin/nifi.sh start`, and your local instance will be running over `https` with the ability
to login via username and password.

The overall work for this improvement was done through [NIFI-8220](https://issues.apache.org/jira/browse/NIFI-8220) and required three major pieces:

* Automatic generation of a self-signed certificate
* Single User Login Identity Provider
* Single User Authorizer

From a high level, the overall setup looks like the following:

<img src="{{ BASE_PATH }}/assets/images/nifi-secure-by-default/01-overview.png" class="img-responsive">

### Automatic Certificate Generation

In order to have any form of authentication & authorization, we first need to be connecting over `https`,
which means NiFi's web server needs a keystore and truststore.

In order to achieve this, [NIFI-8403](https://issues.apache.org/jira/browse/NIFI-8403) introduced the ability to
generate a self-signed certficate during start-up. When keystore and truststore files are specified in `nifi.properties` and the
files don't exist, they will automatically be generated and `nifi.properties` will be updated with the passwords.

As a result, the default `nifi.propeties` file now comes with provided values for the keystore and truststore:

```
nifi.security.keystore=./conf/keystore.p12
nifi.security.keystoreType=PKCS12
nifi.security.keystorePasswd=
nifi.security.keyPasswd=
nifi.security.truststore=./conf/truststore.p12
nifi.security.truststoreType=PKCS12
nifi.security.truststorePasswd=
```

In addition, the default web host and port have been switched to the following `https` values:

```
nifi.web.https.host=127.0.0.1
nifi.web.https.port=8443
```

As a side note, there are two other new properties related to certificates:

```
nifi.security.autoreload.enabled=false
nifi.security.autoreload.interval=10 secs
```

These were not required for the default secure setup, but they allow the keystore and truststore to be reloaded while the application is running. This can be helpful for replacing certificates that may be close to expiring.

### Single User Login Identity Provider

The next step was to provide a mechanism for authenticating the user. NiFi supports many different authentication mechanisms, but most of them require additional dependencies and/or configuration.

In this case, we want a user to login with a username and password without doing anything else. In order to achieve this, [NIFI-8363](https://issues.apache.org/jira/browse/NIFI-8363) introduced the *Single User Login Identity Provider*.

This login identity provider allows a single username/password pair to be configured. When this provider is initialized, if the
username and password are not present, random values will be generated and `login-identity-providers.xml` will be updated with
the values.

The default `login-identity-providers.xml` now contains the following configuration:

```
<provider>
   <identifier>single-user-provider</identifier>
   <class>org.apache.nifi.authentication.single.user.SingleUserLoginIdentityProvider</class>
   <property name="Username"></property>
   <property name="Password"></property>
</provider>
```

NOTE: The password value here is the hashed password. See the last section below about obtaining and changing the default values.

The default `nifi.properies` then specifies this login identity provider:

```
nifi.security.user.login.identity.provider=single-user-provider
```

### Single User Authorizer

The next step was providing a mechanism to perform authorization. In this case, we just want the default user to be authorized for all actions. In order to achieve this, [NIFI-8363](https://issues.apache.org/jira/browse/NIFI-8363) introduced the *Single User Authorizer*.

This authorizer just returns true for all authorization checks, with the caveat that it can only be used when the *Single User Login Identity Provider* is also configured.

The default `authorizers.xml` now contains the following configuration:

```
<authorizer>
   <identifier>single-user-authorizer</identifier>
   <class>org.apache.nifi.authorization.single.user.SingleUserAuthorizer</class>
</authorizer>
```

The default `nifi.properties` then specifies this authorizer:

```
nifi.security.user.authorizer=single-user-authorizer
```

### Default Username/Password

The first time the application is started, the *Single User Login Identity Provider* generates the username and
password and logs them to `nifi-app.log`. An example would be the following:

```
2021-07-16 15:46:31,006 INFO [main] o.a.n.w.c.ApplicationStartupContextListener Flow Controller started successfully.
2021-07-16 15:46:31,026 INFO [main] o.a.n.a.s.u.SingleUserLoginIdentityProvider

Generated Username [6fcaba96-5445-4835-822f-e004c4642d3b]
Generated Password [ScCULiVSEwlqVLG6aHxGv/utRTHxWa7n]

2021-07-16 15:46:31,026 INFO [main] o.a.n.a.s.u.SingleUserLoginIdentityProvider Run the following command to change credentials: nifi.sh set-single-user-credentials USERNAME PASSWORD
2021-07-16 15:46:31,338 INFO [main] o.a.n.a.s.u.SingleUserLoginIdentityProvider Updating Login Identity Providers Configuration [./conf/login-identity-providers.xml]
```

If you then access `https://localhost:8443/nifi` in your browser (accept warnings about
self-signed certificates), you should be able to login with the username/password.

<img src="{{ BASE_PATH }}/assets/images/nifi-secure-by-default/02-nifi-ui-logged-in.png" class="img-responsive">

As the logs mention above, the default username/password can be changed by running the following utility:

```
./bin/nifi.sh set-single-user-credentials USERNAME PASSWORD
```
