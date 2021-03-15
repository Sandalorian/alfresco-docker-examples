Added the repository properties to allow authentication to Alfresco Identity Service:

```yaml
-Dauthentication.chain=identity-service1:identity-service,alfrescoNtlm1:alfrescoNtlm

-Didentity-service.enable-basic-auth=true
-Didentity-service.authentication.validation.failure.silent=false
-Didentity-service.auth-server-url=http://acs1:8888/auth
-Didentity-service.realm=alfresco
-Didentity-service.resource=alfresco
```

Hardcoded URL 

-identity-service.auth-server-url=http://acs1:8888/auth

The hostname and port need to be resolvable from inside the repo container and from the client machine running the browser.

In my lab, I need to access the desktop sync service outside of the local machine. This required the following to be externalized:

-Ddsync.service.uris=http://acs1:9090/alfresco


Docker depends_on does not wait for the dependent container's service to be up and available. This causes alfresco-content-repository container to fail because it will need to pull the well-known configuration from Identity Service. To get around this we will use [wait-for-it](https://github.com/vishnubob/wait-for-it) and override the image's command as follows:

```yaml
        volumes: 
            - ./repository/extension/custom-log4j.properties:/usr/local/tomcat/shared/classes/alfresco/extension/custom-log4j.properties
            - ./repository/tomcat/wait-for-it.sh:/usr/local/tomcat/wait-for-it.sh
        depends_on: 
            - auth
        command: ["./wait-for-it.sh", "auth:8888", "--timeout=0", "--strict", "--", "catalina.sh", "run", "-security"]
```

You will see the alfresco container waiting for the auth container to become available. The log traces will look like the following

```log
alfresco_1            | wait-for-it.sh: waiting for auth:8888 without a timeout
alfresco_1            | wait-for-it.sh: auth:8888 is available after 49 seconds
```

For some reason the realm is not created in the docker images on quay.io. You can import a realm as per [Keycloaks documentation](https://hub.docker.com/r/jboss/keycloak/). I download the realm confirmation from the standalone deployment and then manually imported using the follow environment variable:

```yaml
KEYCLOAK_IMPORT: /tmp/alfresco-realm.json
```
This file is mounted using the follow Docker volume:

```yaml
volumes: 
    - ./identity/alfresco-realm.json:/tmp/alfresco-realm.json
```

Create multiple database using the following example:

https://github.com/mrts/docker-postgresql-multiple-databases