# ACS Docker Compose with mTLS for Search Services communication Example

This is an example of how to configure ACS and Search Services with mTLS authentication. 

This example uses ACS 6.2.2 and Search Service 2.0+. Things are changing with ACS 7 and beyond to secure the key and trust store passwords. Make sure you review the official documentation for the product version you are using to understand the exact steps. 

This is based on the steps outlined in the [Search Services documentation](https://docs.alfresco.com/search-services/latest/install/options/#install-with-mutual-tls-zip).

**This is not a production ready deployment and serves only as an example.**.

## Usage

You will need to make sure you generate trust and key stores as per Alfresco's official documentation. Once generated place the files in the correct location as described in [Generate new key and trust stores](#Generate-new-key-and-trust-stores). 

You will need Docker and Docker-compose setup and functioning beforehand.

1. Clone this Github repository

   `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

1. Make sure you [Generate new key and trust stores](#Generate-new-key-and-trust-stores)
1. Switch into the base directory `cd alfresco-docker-examples/mTLS`
1. [Enable and Configure Secure Communication](#Enable-and-Configure-Secure-Communication)
1. Run `docker-compose up -d`

    This will run the service in detached mode. To tail the logs afterwards, you can use `docker-compose logs -f`

You can now access the ACS resources as on the usual URLs (for example: `http://localhost:8080/share`)

To stop the service run `docker-compose down`

### Changing Versions

You can change the versions of Alfresco-Content-Repository, Alfresco-Share and Alfresco-Search-Services using the .env file.

# How This Works

## Generate new key and trust stores

The easiest way to do this is using [alfresco-ssl-generator](https://docs.alfresco.com/search-services/latest/config/keys/#generate-secure-keys-for-ssl-communication).

**As we are using ACS 6, you must use `-alfrescoformat classic`**

These can be placed in the following directories.

Service | Local Path | alfresco-ssl-generator Files
---|---|---
alfresco | ./repository/alf_data/keystore | ssl.keystore, ssl.truststore, ssl-keystore-passwords.properties, ssl-truststore-passwords.properties
solr6 | ./solr6/solrhome/keystore | ssl-repo-client.keystore, ssl-repo-client.truststore, ssl-keystore-passwords.properties, ssl-truststore-passwords.properties

## Enable and Configure Secure Communication

In the below example property configuration, the passwords used are `keystore` and `truststore`. If you are using different passwords, you must update the below example configurations to match your passwords.

Both the repository and solr will need to be configured as follows:

### Repository

The following properties must be set for the `alfresco/alfresco-content-repository` container:


```yaml
-Dsolr.port.ssl=8983
-Dsolr.secureComms=https

-Dencryption.ssl.keystore.location=/usr/local/tomcat/alf_data/keystore/ssl.keystore
-Dencryption.ssl.keystore.type=JCEKS
-Dencryption.ssl.keystore.keyMetaData.location=/usr/local/tomcat/alf_data/keystore/ssl-keystore-passwords.properties
-Dencryption.ssl.truststore.location=/usr/local/tomcat/alf_data/keystore/ssl.truststore
-Dencryption.ssl.truststore.type=JCEKS
-Dencryption.ssl.truststore.keyMetaData.location=/usr/local/tomcat/alf_data/keystore/ssl-truststore-passwords.properties
```

### SOLR

We use the environment variables to setup properties for the `alfresco/alfresco-search-services` container:

```yaml
- ALFRESCO_SECURE_COMMS=https

- SOLR_SSL_KEY_STORE=/opt/alfresco-search-services/solrhome/keystore/ssl.repo.client.keystore
- "SOLR_SSL_KEY_STORE_PASSWORD=keystore"
- SOLR_SSL_KEY_STORE_TYPE=JCEKS
- SOLR_SSL_TRUST_STORE=/opt/alfresco-search-services/solrhome/keystore/ssl.repo.client.truststore
- "SOLR_SSL_TRUST_STORE_PASSWORD=truststore"
- SOLR_SSL_TRUST_STORE_TYPE=JCEKS
- SOLR_SSL_NEED_CLIENT_AUTH=true
- SOLR_SSL_WANT_CLIENT_AUTH=false
- SOLR_OPTS=-Dsolr.ssl.checkPeerName=false -Dsolr.allow.unsafe.resourceloading=true -Dssl-keystore.password=keystore -Dssl-keystore.aliases=ssl-alfresco-ca,ssl-repo-client -Dssl-keystore.ssl-alfresco-ca.password=keystore -Dssl-keystore.ssl-repo-client.password=keystore -Dssl-truststore.password=truststore -Dssl-truststore.aliases=ssl-alfresco-ca,ssl-repo,ssl-repo-client -Dssl-truststore.ssl-alfresco-ca.password=truststore -Dssl-truststore.ssl-repo.password=truststore -Dssl-truststore.ssl-repo-client.password=truststore
```

When the alfresco/alfresco-search-services image is deploy, two cores are created. These two cores, alfreso and archive, are created using the norerank template. By default this template has secure communication disabled. To save having to modify the cores after creation, we supply a modified norerank template which has secure communication enabled. This modified file has the following configuration changes:

```java
alfresco.secureComms=https

# ssl has been enabled as follows
alfresco.encryption.ssl.keystore.type=JCEKS
alfresco.encryption.ssl.keystore.provider=
alfresco.encryption.ssl.keystore.location=/opt/alfresco-search-services/solrhome/keystore/ssl.repo.client.keystore
alfresco.encryption.ssl.keystore.passwordFileLocation=
alfresco.encryption.ssl.truststore.type=JCEKS
alfresco.encryption.ssl.truststore.provider=
alfresco.encryption.ssl.truststore.location=/opt/alfresco-search-services/solrhome/keystore/ssl.repo.client.truststore
alfresco.encryption.ssl.truststore.passwordFileLocation=
```

### Tomcat

The default Tomcat configuration does not ship with a secure port configured. To so this we modify a out of the box server.xml file and add the following connector:

```xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
    SSLEnabled="true" maxThreads="150" scheme="https"
    keystoreFile="/usr/local/tomcat/alf_data/keystore/ssl.keystore"
    keystorePass="keystore" keystoreType="JCEKS"
    secure="true" connectionTimeout="240000"
    truststoreFile="/usr/local/tomcat/alf_data/keystore/ssl.truststore"
    truststorePass="truststore" truststoreType="JCEKS"
    clientAuth="want" sslProtocol="TLS" />
```