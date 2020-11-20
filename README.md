# ACS Docker Cluster Setup

This is an example of how to setup a cluster using Alfresco Content Services Docker-Compose deployment.

This builds on the [official documentaion](https://docs.alfresco.com/6.2/concepts/ha-intro.html).

There are three components that are setup in this example cluster:
    1. [Repository](#Repository-Cluster-Setup)
    2. [Share](#Share-Cluster-Setup)
    3. [Solr](#Solr-Cluster-Setup)

**This is not a production ready deployment and serves only as an example**

## Repository Cluster Setup

The minimum requirements for clustering are the following:
	1. [Database needs to be shared](#Database-needs-to-be-shared)
	2. [Content Store directory needs to be shared](#Content-Store-directory-needs-to-be-shared)
	3. [License needs to have clustering enabled](#License-needs-to-have-clustering-enabled)
    4. [Creating another node](#Creating-another-node)

We cover each of these in turn below:

### Database needs to be shared

Our docker-compose configuration runs the database as a service called posgres. The db service configuration parameters can be shared with other nodes in the cluster which keeps things simple.

### Content Store directory needs to be shared

By default, in a docker-compose configuration, the Content Store (the location where .bin files are stored) is not share between containers. Docker-compose named volumes would be a good solution.

The process to create this shared Content Store location is:
	1. Declare a volume in your docker-compose.yml
	```ymal
	volumes:
	    shared-file-store-volume:
	        driver_opts:
	            type: tmpfs
	            device: tmpfs
	    content-store:
	```
	2. Add this volume to the alfresco service configuration
    ```yaml
	        volumes: 
	            - content-store:/usr/local/tomcat/alf_data
    ```

When using docker-compose and named volumes, you do need to make sure to delete these volumes, either using `docker-compose down -v` or manually using `docker volume prune`

### License needs to have clustering enabled

Out of the box, the container contains a two day license which does allow clustering. This will cover this demo. More licenses can be purchased if required.

### Creating another node

With Docker and Docker-Compose adding clustering is very close to copy and paste. To get the ball rolling:
	1. Select and copy the whole alfresco service configuration
	2. Paste this directly below the added volume configuration from above (make sure you correct any whitespace misalignment as the yaml file format uses whitespace indentation to denote structure)
	3. Rename the service from `alfresco` to `alfresco-node2`

The official Alfresco recommendation is to start cluster nodes in a rolling start. Docker-compose version 2 does not support anything more advanced than depends_on. Add this to end of the alfresco-node2 service:
    ```yaml
        depends_on: 
            - alfresco
    ```

This gives the alfresco_node2 service just enough of a pause to allow the alfresco service to connect to the database and begin creating the schema.

## Share Cluster Setup

It is possible to configure Share as a cluster of nodes, this isn't strictly required. In the docker world you would need to create a custom share container which contains the required configuration inside a custom-slingshot-application-context.xml file. 

For the purpose of this demo, we will just add another Share container and hardwire the Share containers to their respective Alfresco containers. To do this within docker-compose.yml:
	1. Add a ports section to share service as follows:
	```yaml
	        ports:
	            - 9091:8080
	```
	2. Copy the share service configuration and paste this directly below it's current location
	3. Rename the copied `share` service to `share-node2`
	4. Change the REPO_HOST to match the second repository's service name (alfresco-node2 in this demo). For example:
    ```yaml
	            REPO_HOST: "alfresco-node2"
    ```
	5. change the host port to 9092 as follows:
	```yaml
	        ports: 
	            - 9092:8080
    ```
	6. We must now hardwire alfresco-node2 to share-node by updating it's -Dshare.host value as follows:
    ```
	                -Dshare.host=share_node2
	```

## Solr Cluster Setup

There are a multitude of methods to creating clusters in solr. As with the Share clustering configuration, we will keep things simple. We will hardwire each repository node to an index node and vice versa. To do this:

	1. Create a new solr service by copying and pasting the existing solr6 service. Call this service solr6-node2
	2. Update solr6-node2 so this has the correct environment variables set:
    ```yaml
	            - SOLR_ALFRESCO_HOST=alfresco-node2

	            - SOLR_SOLR_HOST=solr6-node2
	```
	3. Update alfresco-node2 so this points to the correct solr node:
    ```yaml
	                -Dsolr.host=solr6-node2
    ```

Summary

With the above we have a really basic cluster configuration. There are some additional pieces that can be added here:
	1. **Load balancers for clients**
	This would split the load between the "front end" Share containers and ensure if one of them fails, the requests are correctly routed
	2. **Load balancers for the Repository**
	These would allow some reliance if a Repository container was downed or lost. This would require adjusting the configuration so the load balancer is used rather than a hardwire between tiers
	3. **Load balancer for SOLR search requests**
	In the configuration above, each solr node keeps it's own copy of the index. These solr nodes can therefore be accessed via a load balancer in case one is downed or lost.
