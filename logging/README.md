# ACS Docker Compose Logging Example

This is an example of how to configure debug classes Alfresco Content Services.

There is no real documentation for this. Google has plenty of information on available classes and debugging approaches. 

**This is not a production ready deployment and serves only as an example.**.

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

1. Clone this Github repository

   `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

1. Switch into the base directory `cd alfresco-docker-example/logging`
1. Edit `repository/extension/custom-log4j.properties` and `share/web-extension/custom-log4j.properties` with the debug classes you need enabled.
1. Run `docker-compose up -d`

    This will run the service in detached mode. To tail the logs afterwards, you can use `docker-compose logs -f`

You can now access the ACS resources as on the usual URLs (for example: `http://localhost:8080/share`)

To stop the service run `docker-compose down`