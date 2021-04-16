# ACS and Alfresco Identity Service (AIMS) Docker Setup

This is an example of how to setup a ACS with AMIS using [Alfresco Content Services Docker-Compose deployment](https://docs.alfresco.com/content-services/latest/install/containers/docker-compose/).

This builds on the [official documentation](https://docs.alfresco.com/identity-service/latest/tutorial/sso/saml/).

This is a really basic example. You can only access Share and Digital workspace using the built in admin user. There is no ldap sync configure.

**This is not a production ready deployment and serves only as an example.** Please make sure you review the [Food for Thought](#Food-for-thought) below.

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

This does use Enterprise images that can only be accessed via quay.io credentials. You can read more about this [here](https://docs.alfresco.com/content-services/latest/install/containers/docker-compose/) (step 4).

1. Clone this Github repository

    `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

1. Export the Docker host machines hostname to an environment variable name DOCKER_HOST_HOSTNAME. For example:

    On Linux:

    `export DOCKER_HOST_HOSTNAME=$HOSTNAME`

    On Windows:

    `setx DOCKER_HOST_HOSTNAME %COMPUTERNAME%`

    Then close and reopen your command prompt window.

1. Switch into the base directory `cd alfresco-docker-example/identity-service/basic`
1. Run `docker-compose up -d`

You can now access resources on the usual URLs. There is only one account configured:

Username | Password 
--- | --- 
admin | admin

Both Share and ADW are configured to redirect directly to AIMS.

`http://<<Your docker host hostname>>:8080/share `
`http://<<Your docker host hostname>>:8080/workspace`

You can also access AIMS on the following url:

`http://<<Your docker host hame>>:8888/`

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
5. [Configuring ADW to use AIMS](#Configuring-ADW-to-use-AIMS)

## Configuring additional databases

Postgres gives you a few options to create database on creation of your docker container. In this docker compose example, we have modified [docker-postgressql-multiple-databases's](https://github.com/mrts/docker-postgresql-multiple-databases) script to create a database, user and set permissions on multiple databases.

The environment variables used by this script are:

```yaml
- POSTGRES_MULTIPLE_DATABASES=alfresco,keycloak
```

Which creates an alfresco user and database, and a keycloak user and database.


## Adding and configuring the AIMS container

For AIMS to work, this must point to a database. As AIMS is based on Keycloak we can take advantage of the environment variables listed on Keycloaks docker hub page. These are the settings we configure in our docker-compose.yml file:

```yaml
KEYCLOAK_USER: admin
KEYCLOAK_PASSWORD: admin
DB_VENDOR: postgres
DB_ADDR: "postgres:5432"
DB_DATABASE: keycloak
DB_USER: keycloak
DB_PASSWORD: keycloak
KEYCLOAK_IMPORT: /tmp/alfresco-realm.json
JAVA_OPTS: "-Djboss.socket.binding.port-offset=808 -Djava.net.preferIPv4Stack=true -Djava.net.preferIPv4Addresses=true"
```

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

## Configuring ADW to use AIMS

Configuring ADW to use AIMS is well documented [here](https://docs.alfresco.com/identity-service/latest/tutorial/sso/saml/#step-8-configure-alfresco-digital-workspace). We apply the specified environment variables into our docker service for ADW:

```yaml
APP_CONFIG_AUTH_TYPE: OAUTH
APP_CONFIG_OAUTH2_HOST: http://${DOCKER_HOST_HOSTNAME}:8888/auth/realms/alfresco 
APP_CONFIG_OAUTH2_CLIENTID: alfresco
APP_CONFIG_OAUTH2_IMPLICIT_FLOW: 'true'
APP_CONFIG_OAUTH2_SILENT_LOGIN: 'true'
APP_CONFIG_OAUTH2_REDIRECT_SILENT_IFRAME_URI: http://${DOCKER_HOST_HOSTNAME}:8080/workspace/assets/silent-refresh.html
APP_CONFIG_OAUTH2_REDIRECT_LOGIN: /workspace/
APP_CONFIG_OAUTH2_REDIRECT_LOGOUT: /workspace/logout
```

# Food for thought

There is only one client configured in AIMS, `alfresco`. In some instances you may want to separate Share, ADW, Repo/ DSync, in to different clients.

You must specify the environment variable `DOCKER_HOST_HOSTNAME` otherwise this example will not work. This should also be the name of the host that is used to access any of the web applications.

There have been instances where the repository image will not be able to correctly resolve DOCKER_HOST_HOSTNAME, in this case it is likely that the docker host's /etc/resolv.conf is miss configured. You will need to correct this, typically a reboot will do, and then bring up the service again. 