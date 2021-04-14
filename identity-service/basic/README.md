# ACS and Alfresco Identity Service (AIMS) Docker Setup

This is an example of how to setup a ACS with AMIS using [Alfresco Content Services Docker-Compose deployment](https://docs.alfresco.com/content-services/latest/install/containers/docker-compose/).

This builds on the [official documentation](https://docs.alfresco.com/identity-service/latest/tutorial/sso/saml/).

**This is not a production ready deployment and serves only as an example.** Please make sure you review the [Food for Thought](#Food-for-thought) below.

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

This does use Enterprise images that can only be accessed via quay.io credentials. You can read more about this [here](https://docs.alfresco.com/content-services/latest/install/containers/docker-compose/) (step 4).

1. Clone this Github repository

    `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

1. Export the Docker host machines hostname to an environment variable name DOCKER_HOST_HOSTNAME. For example:

    On Linux:

    `export $DOCKER_HOST_HOSTNAME=$HOSTNAME`

    On Windows:

    `setx DOCKER_HOST_HOSTNAME %COMPUTERNAME%`

    Then close and reopen your command prompt window.

1. Switch into the base directory `cd alfresco-docker-example/identity-service/basic`
1. Run `docker-compose up -d`

You can now access resources on the usual URLs

To stop the service and remove it's volumes run `docker-compose down -v`

### Changing Versions

You can change the versions of Alfresco-Content-Repository, Alfresco-Share and Alfresco-Search-Services using the .env file.

# How this works

The best resource I've found for understanding how ACS + AIMS works in any details can be found [here](https://hub.alfresco.com/t5/alfresco-content-services-blog/deploying-alfresco-dbp-with-identity-service-using-docker/ba-p/299338). This Docker compose setup is actually based on this. 

These are the main parts to make this work:

1. [Configuring additional databases](#Configuring-additional-databases)
2. [Adding the AIMS container](#Adding-the-AIMS-container)
3. [Configuring ACS to use AIMS](#Configuring-ACS-to-use-AIMS)
4. [Configuring Share to use AIMS](#Configuring-Share-to-use-AIMS)

## Configuring additional databases

Create multiple database using the following example:

https://github.com/mrts/docker-postgresql-multiple-databases

## Adding and configuring the AIMS container

TODO: database configuration details

### Adding an alfresco realm

For some reason the realm is not created in the docker images on quay.io. You can import a realm as per [Keycloaks documentation](https://hub.docker.com/r/jboss/keycloak/). I download the realm confirmation from the standalone deployment and then manually imported using the follow environment variable:

```yaml
KEYCLOAK_IMPORT: /tmp/alfresco-realm.json
```
This file is mounted using the follow Docker volume:

```yaml
volumes: 
    - ./identity/alfresco-realm.json:/tmp/alfresco-realm.json
```

## Configuring ACS to use AIMS

This docker compose examples works by using Alfresco's docker image of it's identity service. 

Added the repository properties to allow authentication to Alfresco Identity Service:

```yaml
-Dauthentication.chain=identity-service1:identity-service,alfrescoNtlm1:alfrescoNtlm

-Didentity-service.enable-basic-auth=true
-Didentity-service.authentication.validation.failure.silent=false
-Didentity-service.auth-server-url=http://'${DOCKER_HOST_HOSTNAME}':8888/auth
-Didentity-service.realm=alfresco
-Didentity-service.resource=alfresco
```

### Delaying alfresco-content-repository start up

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

## Configuring Share to use AIMS

Configuring Share with AIMS is as simple as setting some additional Java options. In this deployment we use the following settings take from the [official documentation](https://docs.alfresco.com/identity-service/latest/tutorial/sso/saml/#step-9-configure-alfresco-share-properties):

```yaml
-Daims.enabled=true
-Daims.realm=alfresco
-Daims.resource=alfresco
-Daims.authServerUrl=http://'${DOCKER_HOST_HOSTNAME}':8888/auth
-Daims.publicClient=true
```

# Food for thought





