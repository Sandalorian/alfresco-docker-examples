# ACS Docker with LDAP to Microsoft Active Directory

This is an example of how to deploy Alfresco Content Services and sync users and groups using LDAP from Microsoft Active Directory.

This builds on the [official documentation](https://docs.alfresco.com/content-services/latest/admin/auth-sync/#configure-ldap).

**This is not a production ready deployment and serves only as an example.**.

The example assume that you already have an AD forest setup, with users and groups. If you do not, have a look at my Vagrant machine for automating this setup [here](https://github.com/sirReeall/vagrant_machines/tree/master/ActiveDirectory/Windows-2016-std). If you are not using my Vagrant machine, make sure you customise the properties defined below in [Configure ACS LDAP Sync and Authentication](#Configure-ACS-LDAP-Sync-and-Authentication)

## Usage

You will need Docker and Docker-compose setup and functioning beforehand.

1. Clone this Github repository

   `git clone https://github.com/sirReeall/alfresco-docker-examples.git`

2. Switch into the base directory `cd alfresco-docker-example/ldap-ad`
3. Run `docker-compose up -d`

You can now access the ACS resources as on the usual URLs (for example: `http://localhost:8080/share`)

To stop the service run `docker-compose down`

### Configure ACS LDAP Sync and Authentication

The following key properties must be know to make this work:

Property | Example
--- | --- 
Domain Name | example.com
IP address and port of AD server | 182.168.100.25:389
AD account used for sync | alfresco.ldap@example.com
Password for AD account used fot sync | C0mplexPassword
DN of User search base | OU=Alfresco-Users,DC=example,DC=com
DN of Group search base | OU=Alfresco-Groups,DC=example,DC=com

Add the following to the JAVA_OPTS environment variable section in your docker-compose.yml file:

```yaml
-Dldap.authentication.allowGuestLogin=true
-Dldap.authentication.userNameFormat=%s@example.com
-Dldap.authentication.java.naming.provider.url=ldap://192.168.100.25:389
-Dldap.authentication.defaultAdministratorUserNames=Administrator
-Dldap.synchronization.java.naming.security.principal=alfresco.ldap@example.com
-Dldap.synchronization.java.naming.security.credentials=C0mplexPassword
-Dldap.synchronization.groupSearchBase=OU=Alfresco-Groups,DC=example,DC=com
-Dldap.synchronization.userSearchBase=OU=Alfresco-Users,DC=example,DC=com

-Dauthentication.chain=alfrescoNtlm1:alfrescoNtlm,ldap1:ldap-ad
```

### Resolve domain name

In some cases it will be necessary to add host entries into the repository service so that the domain controller can be resolved. To do this we add the following extra_hosts sections into docker-compose.yml, where `dc1` is the Microsoft AD server name:

```yaml
extra_hosts:
    - "dc01.example.com:192.168.100.25"
    - "dc01:192.168.100.25"
```

### Changing Versions

You can change the versions of Alfresco-Content-Repository, Alfresco-Share and Alfresco-Search-Services using the .env file.
