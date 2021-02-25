# ACS Docker with MySQL

This is an example of how to setup Alfresco Content Services Docker-Compose deployment with MySQL.

This builds on the [official documentation](https://docs.alfresco.com/content-services/latest/config/databases/#mysql-and-mariadb).

**This is not a production ready deployment and serves only as an example.** Please make sure you review the [Food for Thought](#Food-for-thought) below.

# Food for thought

## Docker best Practices

The idea behind this repository is to quick be able to spin up ACS with a MySQL backend. The JDBC driver is installed my mounting the jar as a volume rather then building custom Docker images. This isn't the worst practice, but you would want to make sure that you understand how volumes are being used before using this example anywhere else. Typically, you would create a custom image with the driver already installed.
