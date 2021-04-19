# ACS Docker with a custom model

This is an example of how to deploy Alfresco Content Services with a bootstraped content model using Docker-compose.

This builds on the [official documentation](https://docs.alfresco.com/content-services/6.0/develop/repo-ext-points/content-model/).

The model deployed is based on the [acme document model sample](https://github.com/Alfresco/alfresco-sdk/blob/sdk-4.2/archetypes/alfresco-platform-jar-archetype/src/main/resources/archetype-resources/src/main/resources/alfresco/module/__artifactId__/model/content-model.xml) that is included with the Alfresco SDK.

**This is not a production ready deployment and serves only as an example.** Please make sure you review the [Food for Thought](#Food-for-thought) below.

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

1. Clone this Github repository

   `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

2. Switch into the base directory `cd alfresco-docker-examples/content-modeling`
3. Run `docker-compose up -d`

You can now access the ACS resources as on the usual URLs.

To stop the service run `docker-compose down`

### Create an Acme Document Type

You can find the model that is deployed in the `repository/extension` folder.

You can create nodes of type acme:document using any of the Alfresco Public APIs. Below are some examples:

#### REST API

You can create nodes of type acme:document and set acme:documentId using the following curl command:

```bash
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Authorization: Basic YWRtaW46YWRtaW4=' -d '{ "name":"My acme doc", "nodeType":"acme:document",  "properties": { "acme:documentId": "A1234" } }' 'http://localhost:8080/alfresco/api/-default-/public/alfresco/versions/1/nodes/-my-/children'
```

### Searching for an Acme Document Type

You can then search for this node using acme:documentId as follows:

**TMDQ**

As acme:documentId is of type d:text, we must prefix the search query with an equals sign as per the [documentation](https://docs.alfresco.com/insight-engine/latest/config/transactional/#alfresco-fts-ql--tmdq).

```bash
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Authorization: Basic YWRtaW46YWRtaW4=' -d '{ "query": { "query": "=acme:documentId:A1234" } }' 'http://localhost:8080/alfresco/api/-default-/public/search/versions/1/search'
```

**SOLR**

```bash
curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Authorization: Basic YWRtaW46YWRtaW4=' -d '{ "query": { "query": "=acme:documentId:A1234" } }' 'http://localhost:8080/alfresco/api/-default-/public/search/versions/1/search'
```

## Food for Thought

This example directly mounts the content model xml definitions into the repository. It is debatable if this is a good or bad idea, and there are multiple approaches to this (custom container, custom amp, etc). This is only one example, you should make sure you are familiar with the approach you choose in your environment.
