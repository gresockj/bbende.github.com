---
layout: post
title: "Apache NiFi 1.15.0 - Protecting NiFi Configuration using HashiCorp Vault K/V Secrets"
description: ""
author: Joe Gresock
category: "Development"
tags: [NiFi,Vault]
---
{% include JB/setup %}

#### (Guest author: Joe Gresock)

This is a follow-on post to [Apache NiFi 1.14.0 - HashiCorp Vault Integration](https://bryanbende.com/development/2021/07/20/apache-nifi-1-14-0-hashicorp-vault), demonstrating protecting Apache NiFi 1.15.0 sensitive configuration properties by storing them as HashiCorp Vault Key/Value Secrets.

### Introduction

In the [last post](https://bryanbende.com/development/2021/07/20/apache-nifi-1-14-0-hashicorp-vault), we walked through protecting NiFi configuration properties using HashiCorp Vault's Transit Secrets Engine.  This process outsourced the encryption of sensitive configuration properties to Vault.  Here we will slightly alter the procedure in order to store these sensitive properties as actual Key/Value Secrets in a Vault server.

Rather than simply pointing out the differences, we'll walk through the entire procedure, which will be necessary in order to pick up the latest Apache NiFi distribution anyway.

### Setup

To get started, we'll need to download and install HashiCorp Vault, NiFi 1.15.0, and nifi-toolkit.
* First, follow the relevant Vault installation guide for your system: [https://www.vaultproject.io/downloads](https://www.vaultproject.io/downloads)
* Then head over to the Apache nifi downloads page: [https://nifi.apache.org/download.html](https://nifi.apache.org/download.html)
  * Download `nifi-toolkit-1.15.0-bin.zip`
  * Download `nifi-1.15.0-bin.zip`

For the purposes of this guide, we'll unzip both `nifi-toolkit-1.15.0-bin.zip` and `nifi-1.15.0-bin.zip` in the same parent directory, so that you should see this:
```
$ ls
nifi-1.15.0    nifi-toolkit-1.15.0
```

We'll start Vault using development mode, which allows us to easily interact with the Vault server without having to [unseal](https://www.vaultproject.io/docs/concepts/seal) the server.  Run the following:
```
vault server -dev > init.log
```
This will launch Vault in development mode, and will write something like the following to init.log:
```
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: fUJ6zxqlo+vsWFlCxMK9MKrhgpsmxdNla727mu+5nuY=
Root Token: s.3qsioJdA3YXptvEQURDCijUw

Development mode should NOT be used in production installations!
```

In another terminal, create a file containing this root token, which we'll use later:
```
TOKEN=$(grep "Root Token" init.log | cut -d ":" -f2 | xargs)
echo "vault.token=$TOKEN" > ~/vault-auth.properties
```

Run the following to enable the K/V (version 1) Secrets Engine, which actually stores sensitive values in the Vault server:
```
vault secrets enable -path nifi-kv kv
```

This last command created a Vault path called `"nifi-kv"`.  Take note of this value, because we'll be seeing it again later.  Note that we could have simply used the command `vault secrets enable kv`, which would have enabled the K/V secrets engine at the path `kv`, but here we use a specific path in order to see how it is customized.

### Integrating Vault into nifi and nifi-tookit

Now that we have Vault running and configured, we'll configure nifi-toolkit to talk to Vault.  Since the toolkit operates on actual NiFi configuration files, that's where we'll start.

First, open up `nifi-1.15.0/conf/bootstrap-hashicorp-vault.conf` in your preferred editor, and configure it as follows:
```
# HTTP or HTTPS URI for HashiCorp Vault is required to enable the Sensitive Properties Provider
vault.uri=http://127.0.0.1:8200

# Transit Path is required to enable the Sensitive Properties Provider Protection Scheme 'hashicorp/vault/transit/{path}'
vault.transit.path=

# Key/Value Path is required to enable the Sensitive Properties Provider Protection Scheme 'hashicorp/vault/kv/{path}'
vault.kv.path=nifi-kv

# Token Authentication example properties
# vault.authentication=TOKEN
# vault.token=<token value>

# Optional file supports authentication properties described in the Spring Vault Environment Configuration
# https://docs.spring.io/spring-vault/docs/2.3.x/reference/html/#vault.core.environment-vault-configuration
#
# All authentication properties must be included in bootstrap-hashicorp-vault.conf when this property is not specified.
# Properties in bootstrap-hashicorp-vault.conf take precedence when the same values are defined in both files.
# Token Authentication is the default when the 'vault.authentication' property is not specified.
vault.authentication.properties.file=<full/path/to/vault-auth.properties>
```

Make sure you've specified the full path to the `vault-auth.properties` file that you created when installing Vault.  Notice that we could have actually stored the `vault.token` directly in the `bootstrap-hashicorp-vault.conf` file, but it's considered better practice to keep your authentication properties separate, in case you decide to rotate your Vault authentication token or other credentials.

At this point, we've integrated Vault into NiFi. Now copy this configuration file into nifi-toolkit and we'll be ready to start encrypting files:
```
cp nifi-1.15.0/conf/bootstrap-hashicorp-vault.conf nifi-toolkit-1.15.0/conf/
```

### Encrypting files using the HASHICORP_VAULT_KV protection scheme

We will have some sensitive values to protect in `nifi-1.15.0/conf/nifi.properties` once we start NiFi:

```
cd nifi-1.15.0
./bin/nifi.sh start
tail -f logs/nifi-app.log
```

Once you see the following:
```
2021-10-29 15:12:32,167 INFO [main] org.apache.nifi.web.server.JettyServer NiFi has started. The UI is available at the following URLs:
2021-10-29 15:12:32,167 INFO [main] org.apache.nifi.web.server.JettyServer https://127.0.0.1:8443/nifi
```
Then `Ctrl-C` from tailing the log and stop NiFi:
```
./bin/nifi.sh stop
```

Now there should be some generated passwords in our `nifi-1.15.0/conf/nifi.properties` file.  We can encrypt them using the following commands:
```
cd ../nifi-toolkit-1.15.0
# The breakdown of this command is as follows:
# -b specifies the NiFi bootstrap.conf, which specifies nifi.bootstrap.protection.hashicorp.vault.conf=./conf/bootstrap-hashicorp-vault.conf
# -n specifies the nifi.properties file to encrypt
# -o specifies the location to output the encrypted nifi.properties file.  If we left out this argument, it would simply encrypt nifi.properties in place.
# -S specifies the protection scheme.  If we left out this argument, it would use the default AES_GCM protection scheme.
./bin/encrypt-config.sh -b ../nifi-1.15.0/conf/bootstrap.conf \
    -n ../nifi-1.15.0/conf/nifi.properties \
    -o nifi.properties.encrypted \
    -S HASHICORP_VAULT_KV
```

We're prompted to enter a password (must be at least 12 characters), which will be used to configure the `nifi-1.15.0/conf/bootstrap.conf` with a `nifi.bootstrap.sensitive.key`.  This value is not used by the HashiCorp Vault protection scheme, but is required for other protection schemes.  Since different properties are permitted to be protected by different protection schemes at the same time, the key is still generated from the password you enter.

After you enter a password, you should see output like the following:
```
2021/10/28 16:02:33 INFO [main] org.apache.nifi.properties.NiFiPropertiesLoader: Loaded 202 properties from ../nifi-1.15.0/conf/nifi.properties
2021/10/28 16:02:33 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Loaded NiFiProperties instance with 202 properties
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.keyPasswd with hashicorp/vault/kv/nifi-kv -> 	nifi-kv/default/nifi.security.keyPasswd
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.keyPasswd.protected
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.keystorePasswd with hashicorp/vault/kv/nifi-kv -> 	nifi-kv/default/nifi.security.keystorePasswd
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.keystorePasswd.protected
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.security.truststorePasswd with hashicorp/vault/kv/nifi-kv -> 	nifi-kv/default/nifi.security.truststorePasswd
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.security.truststorePasswd.protected
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Protected nifi.sensitive.props.key with hashicorp/vault/kv/nifi-kv -> 	nifi-kv/default/nifi.sensitive.props.key
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Updated protection key nifi.sensitive.props.key.protected
2021/10/28 16:02:35 INFO [main] org.apache.nifi.properties.ConfigEncryptionTool: Final result: 205 keys including 4 protected keys

```

Our new protected properties can be seen in `nifi.properties.encrypted`:
```
nifi.security.keystore=./conf/keystore.p12
nifi.security.keystoreType=PKCS12
nifi.security.keystorePasswd=nifi-kv/default/nifi.security.keystorePasswd
nifi.security.keystorePasswd.protected=hashicorp/vault/kv/nifi-kv
nifi.security.keyPasswd=nifi-kv/default/nifi.security.keyPasswd
nifi.security.keyPasswd.protected=hashicorp/vault/kv/nifi-kv
nifi.security.truststore=./conf/truststore.p12
nifi.security.truststoreType=PKCS12
nifi.security.truststorePasswd=nifi-kv/default/nifi.security.truststorePasswd
nifi.security.truststorePasswd.protected=hashicorp/vault/kv/nifi-kv
```

Notice that the `.protected` properties indicate `hashicorp/vault/kv/nifi-kv`, which tells NiFi to use the HashiCorp Vault K/V Sensitive Property Provider using a Vault path prefix of `"nifi-kv"`.

Also observe that the actual property values are simply the Vault paths of the relevant secrets.  For example:
```
nifi.security.keystorePasswd=nifi-kv/default/nifi.security.keystorePasswd
```

This tells us that the keystore password is now stored in a Vault Secret named `nifi-kv/default/nifi.security.keystorePasswd`.  Well, let's test it out!

```
vault kv get nifi-kv/default/nifi.security.keystorePasswd
```

This produces output like the following:
```
==== Data ====
Key      Value
---      -----
value    [REDACTED]
```

### Starting NiFi

Now that we've successfully encrypted our `nifi.properties` using HashiCorp Vault, let's start it up and see it decrypt the values.

```
cp nifi.properties.encrypted ../nifi-1.15.0/conf/nifi.properties
cd ../nifi-1.15.0
./bin/nifi.sh start
tail -f logs/nifi-app.logs
```

Since you've already configured `nifi-1.15.0/conf/bootstrap-hashicorp-vault.conf`, NiFi should use that configuration to connect to Vault when it encounters the `hashicorp/vault/kv/nifi-kv` values in `nifi.properties`, allowing it to retrieve the protected properties from the Vault server.  If everything works as planned, you should be able to view NiFi's UI at [https://localhost:8443/nifi](https://localhost:8443/nifi), this time with your properties pulled from the Vault Server at startup.

### Other Secrets Managers

With the Apache NiFi 1.15.0 release also come some new sensitive property providers for Azure KeyVault Secrets (`AZURE_KEYVAULT_SECRET` nifi-toolkit protection scheme) and AWS Secrets Manager (`AWS_SECRETSMANAGER` nifi-toolkit protection scheme).  These providers function very similarly to the one we just saw, storing secrets in their respective cloud provider secrets managers.
