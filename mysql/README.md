# ACS Docker with MySQL

This is an example of how to deploy Alfresco Content Services using MySQL on Docker-Compose.

This builds on the [official documentation](https://docs.alfresco.com/content-services/latest/config/databases/#mysql-and-mariadb).

**This is not a production ready deployment and serves only as an example.** Please make sure you review the [Food for Thought](#Food-for-thought) below.

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

1. Clone this Github repository

   `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

2. Switch into the base directory `cd alfresco-docker-example/mysql`
3. Run `docker-compose up -d`

You can now access the ACS resources as on the usual URLs (for example: `http://localhost:8080/share`)

To stop the service run `docker-compose down`

### Connecting to the MySQL DB

You can connect to the MySQL DB using MySQL Workbench. The connection details are taken from [docker-compose.yml](./docker-compose.yml#L113)

```
Hostname: 127.0.0.1
Port: 3306
Username: root
Password: alfresco
```
### Changing Versions

You can change the versions of Alfresco-Content-Repository, Alfresco-Share and Alfresco-Search-Services using the .env file.

# How this works

There are two main parts to this example:

1. [MySQL configuration](#MySQL-image-and-configuration)
2. [JDBC driver and configuration](#JDBC-driver-and-configuration)

## MySQL image and configuration

The OOTB Postrgresql image is replaced by a MySQL 5.7.32 image. The container is configured to start up with the following configuration:

```yaml
- MYSQL_DATABASE=alfresco
- MYSQL_USER=alfresco
- MYSQL_PASSWORD=alfresco
- MYSQL_ROOT_PASSWORD=alfresco
```

The MySQL image documentation can be found [here](https://hub.docker.com/_/mysql)

## JDBC driver and configuration

The OOTB Alfresco Docker images come with a preinstalled Postgresql JDBC driver. As we are using MySQL in this example, we need to install a MySQL JBDC driver. In my option, the best option for this would be build a custom image, however for simplicity I have used a Docker host volume to mount just the jar file directly into the running container. 

The JDBC is mounted into two containers, alfresco-content-repository and service-sync.

### alfreco-content-repository

We mount the jar as follows:

```yaml
volumes: 
    - ./repository/tomcat/lib/mysql-connector-java-5.1.48.jar:/usr/local/tomcat/lib/mysql-connector-java-5.1.48.jar
```

Then we set the properties to use the jar and the MySQL container:

```yaml
-Ddb.driver=com.mysql.jdbc.Driver
-Ddb.username=alfresco
-Ddb.password=alfresco
-Ddb.url=\"jdbc:mysql://mysql:3306/alfresco?useUnicode=yes&characterEncoding=UTF-8&useSSL=false\"
```
### service-sync

We mount the jar as follows (this is a different location inside the container to alfresco-content-repository):

```yaml
volumes: 
-  ./repository/tomcat/lib/mysql-connector-java-5.1.48.jar:/opt/alfresco-sync-service/connectors/mysql-connector-java-5.1.48.jar
```

Then set the properties:

```yaml
-Dsql.db.driver=com.mysql.jdbc.Driver
-Dsql.db.url=\"jdbc:mysql://mysql:3306/alfresco?useUnicode=yes&characterEncoding=UTF-8&useSSL=false\"
-Dsql.db.username=alfresco
-Dsql.db.password=alfresco
```

# Food for thought

## Docker best Practices

The idea behind this repository is to quick be able to spin up ACS with a MySQL backend. 

The JDBC driver is installed by mounting the jar as a volume rather then building custom Docker images. You would want to make sure that you understand how volumes are being used before using this example anywhere other then a test lab. Typically, you would create a custom image with the driver already installed. this making this image self contained.
