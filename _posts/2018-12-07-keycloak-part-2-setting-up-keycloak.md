---
title: 'Keycloak SSO Part 2: Setting up Keycloak'
date: '07-12-2018 18:35'
categories:
  - 'Blog'
  - 'Security'
tag:
  - 'Keycloak'
---

In part two of this series, we are going to look at setting up a standalone-ha deployment of Keycloak on two CentOS 7 servers.

There are three different deployment types for Keycloak, Standalone, Standalone-HA, and Domain Clustered. Standalone deployments are single servers, this is good for a dev or test environment, but not very useful for production use. Standalone-HA are one or more servers which can both be used to serve authentication requests. This method requires a shared database, and each server is configured manually. In a Domain deployment, there is a master server known as the domain controller, and one or more host controllers which serve authentication requests. This mode allows the host controllers to all have an updated configuration when it is changed on the domain controller, greatly reducing administration overhead with multiple servers.

## Installation

Hardware requirements, as well as distribution directory structure and operation mode information can be found at [https://www.keycloak.org/docs/latest/server_installation/index.html#installation](https://www.keycloak.org/docs/latest/server_installation/index.html#installation)

I use Ansible to deploy the folder structure for Keycloak and make sure all dependencies are setup correctly. This allows me to easily update and ensure that all my Keycloak servers are deployed correctly. My playbook calls [https://github.com/andrewrothstein/ansible-keycloak](https://github.com/andrewrothstein/ansible-keycloak) with some configuration file changes.

## Configuration

Reading and understanding the official documentation is essential to installing Keycloak in a secure manner, I highly recommend you follow the information there and use my configuration as a guide.

### Operation Mode

The first thing to think about when deploying Keycloak is what operation mode you want to use. This will mostly come down to your environment, and the configuration of most modes is the same, just within different files. I am most experienced with Standalone-HA mode, so that is what we are going to work with in this series.

Configuration for this mode is done in the standalone-ha.xml configuration file found at `$keycloak_home/standalone/configuration/standalone-ha.xml`. This file needs to be edited on all servers in a standalone-ha cluster setup.

### Relational Database Setup

The next thing we have to do is setup Keycloak to use a database, since we are going to be creating a deployment with multiple servers, we are going to need a shared database. Setting up a centrally accessible database is beyond the scope of this article. Just know that we are going to be using a PostgreSQL database that is hosted outside of both of the Keycloak servers.

#### Download a JDBC driver

The first step in setting up a database for Keycloak is to download a JDBC driver for the database. This allows Java to interact with the database. You can usually find these on the main site of your chosen database. For example, PostgreSQL's JDBC driver can be found here: https://jdbc.postgresql.org/download.html

#### Package the driver JAR and install

The official documentation is a good resource for how to package the driver for use with Keycloak, and there is no point in duplicating efforts. It can be found here: [https://www.keycloak.org/docs/latest/server_installation/index.html#package-the-jdbc-driver](https://www.keycloak.org/docs/latest/server_installation/index.html#package-the-jdbc-driver)

This boils down to adding a folder structure, copying the .jar file, and adding an .xml file like the following:

```xml
<?xml version="1.0" ?>
<module xmlns="urn:jboss:module:1.3" name="org.postgresql">

    <resources>
        <resource-root path="postgresql-9.4.1212.jar"/> <!-- update the filename to match your PostgreSQL JDBC driver file name -->
    </resources>

    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```

Making sure to update the path with the correct file name.

#### Declare and load the driver

This part, as well as modifying the datasource are a bit more advanced, so I will go over them in a bit more detail here, however the documentation is still very helpful.

We are going to look at the standalone-ha.xml file we were working on earlier, specifically the `drivers` XML block. In this block, we will be adding an additional driver. We can mostly copy the existing format of the h2 driver, and update the information for PostgreSQL. Below is an example of a driver in `standalone-ha.xml`

```xml
<drivers>
    <driver name="h2" module="com.h2database.h2">
        <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
    </driver>
    <driver name="postgresql" module="org.postgresql">
        <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
    </driver>
</drivers>
```

As we can see, the declaration of the driver is nearly identical to the pre-configured H2 database driver.

#### Modify the Keycloak Datasource

Below we will see an example of a working PostgreSQL datasource configuration.

```xml
<datasource jndi-name="java:jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true" use-java-context="true">
    <connection-url>jdbc:postgresql://$URL:$PORT/$DATABASE</connection-url>
    <driver>postgresql</driver>
    <pool>
        <max-pool-size>20</max-pool-size>
    </pool>
    <security>
        <user-name>$USERNAME</user-name>
        <password>$PASSWORD</password>
    </security>
</datasource>
```

$URL = The URL or IP Address of the PostgreSQL server  
$PORT = The port to connect to PostgreSQL database  
$DATABASE = The name of the database that is configured for Keycloak  
$USERNAME = The username that has access to the database specified above  
$PASSWORD = The password of the user defined above  

In the end, you should end up with a `datasources` section that looks like the following:

```xml
<subsystem xmlns="urn:jboss:domain:datasources:5.0">
    <datasources>
        <datasource jndi-name="java:jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true" use-java-context="true">
            <connection-url>jdbc:postgresql://$URL:$PORT/$DATABASE</connection-url>
            <driver>postgresql</driver>
            <pool>
                <max-pool-size>20</max-pool-size>
            </pool>
            <security>
                <user-name>$USERNAME</user-name>
                <password>$PASSWORD</password>
            </security>
        </datasource>
        <drivers>
            <driver name="h2" module="com.h2database.h2">
                <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
            </driver>
            <driver name="postgresql" module="org.postgresql">
                <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
            </driver>
        </drivers>
    </datasources>
</subsystem>
```

## Clustering

The above steps will get a basic setup going with a shared database, however to properly cluster Keycloak, there are a few more steps that need to be completed.

The relevent sections from the Keycloak documentation is below:

1. [Pick an operation mode](https://www.keycloak.org/docs/latest/server_installation/index.html#_operating-mode)

2. [Configure a shared external database](https://www.keycloak.org/docs/latest/server_installation/index.html#_database)

3. [Set up a load balancer](https://www.keycloak.org/docs/latest/server_installation/index.html#_setting-up-a-load-balancer-or-proxy)

4. [Supplying a private network that supports IP multicast](https://www.keycloak.org/docs/latest/server_installation/index.html#multicast-network-setup)

We have already completed steps 1 and 2 in setting up a cluster. There are some additional setup needed for the next two operations, parts of which are made serviceable here, but are covered in more detail at the above links.

### Set up a load balancer

#### Identifying client IP addresses

It is very important that Keycloak is able to identify client IP addresses for various reasons, which are explained further in the docs. We will go over the changes that have to be made in `standalone-ha.xml` here.

You will need to configure the `urn:jboss:domain:undertow:6.0` block to look like below:

```xml
<subsystem xmlns="urn:jboss:domain:undertow:6.0">
   <buffer-cache name="default"/>
   <server name="default-server">
      <ajp-listener name="ajp" socket-binding="ajp"/>
      <http-listener name="default" socket-binding="http" redirect-socket="https" proxy-address-forwarding="true"/>
      ...
   </server>
   ...
</subsystem>
```

#### Enable HTTPS with a Reverse Proxy

If you have a reverse proxy in front of Keycloak which handles your SSL connections and terminations, you need to make the following changes:

In `urn:jboss:domain:undertow:6.0` block (configured above) change the `redirect-socket` from https to a socket binding which we will define.

```xml
<subsystem xmlns="urn:jboss:domain:undertow:6.0">
    ...
    <http-listener name="default" socket-binding="http"
        proxy-address-forwarding="true" redirect-socket="proxy-https"/>
    ...
</subsystem>
```

We will now need to add a new socket binding to the `socket-binding-group` element, like below:

```xml
<socket-binding-group name="standard-sockets" default-interface="public"
    port-offset="${jboss.socket.binding.port-offset:0}">
    ...
    <socket-binding name="proxy-https" port="443"/>
    ...
</socket-binding-group>
```

## Testing the Cluster

Once the changes have been made on all of your Keycloak servers, we can manually start the Keycloak servers in any order. The command for doing so is

```bin/standalone.sh --server-config=standalone-ha.xml``` from the Keycloak home directory. The Keycloak servers will automatically configure themselves if they are connected to the same external database, and you can use your load balancer or reverse proxy to connect to either server to perform authentication operations.

### Firewall

Ensure you have configured the firewall correctly, Keycloak listens on ports 8080 and 8443 by default. There may be additional ports that need to be opened based on your configuration.

## Running at boot

Assuming your tests have passed and you can reach both of your Keycloak servers directly and through your load balancer, you are ready to setup a systemd unit file and have Keycloak start at boot.

Below is a copy of the systemd unit file I am using, which is placed at `/etc/systemd/system/keycloak.service`:

```
[Unit]
Description=Keycloak Identity Provider
After=syslog.target network.target
Before=httpd.service

[Service]
Environment=LAUNCH_JBOSS_IN_BACKGROUND=1 JAVA_HOME=/usr/local/java
User=keycloak
Group=keycloak
LimitNOFILE=102642
PIDFile=/var/run/keycloak/keycloak.pid
ExecStart=/usr/local/keycloak/bin/standalone.sh --server-config=standalone-ha.xml
#StandardOutput=null

[Install]
WantedBy=multi-user.target
```

Once you have completed this step, you can start and enable the service by running the below commands on all of your Keycloak servers:

`systemctl enable keycloak`

`systemctl start keycloak`
