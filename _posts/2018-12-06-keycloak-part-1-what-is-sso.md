---
title: 'Keycloak SSO Part 1: What is single sign on?'
publish_date: '06-12-2018 21:00'
categories:
  - 'Blog'
  - 'Security'
tag:
  - 'Keycloak'
---

In this series, we will take a look at Keycloak, an open-source single sign on solution similar to Microsoft's ADFS product.

In this first part, we will take a brief look at what SSO and Keycloak are, and why they are used.

## What is Single Sign On

Single sign-on (SSO) is a property of access control of multiple related, yet independent, software systems. [Wikipedia](https://en.m.wikipedia.org/wiki/Single_sign-on)

What this means is that we can use a single authentication service to allow users to login to other services, without providing a password to the service that is being logged into.

This is different than connecting each of these applications to LDAP, as this requires providing a username and password to every service that you want to login to.

### Benefits of SSO

1. Central location for authentication allows for easier auditing and security.

2. Central location for configuration of authorization, ie, Jane is able to access email but not the CRM suite

3. Authentication to external services such as a hosted CRM suite are made possible without sending LDAP requests over the internet

4. Tokens are wrapped in SSL/TLS as part of the HTTPS connection, but can also be both signed and/or encrypted using keys known only between the identity provider and service for increased security

## What is Keycloak

Keycloak is an open source program that allows you to setup a secure single sign on provider. It supports multiple protocols such as SAML 2.0 and OpenID Connect. It can also store user credentials locally or via an LDAP or Kerberos backend.

These features allows Keycloak to be highly configurable, but also fairly easy to install and setup.

More information about Keycloak can be found here: https://www.keycloak.org

### Basic Workflow

The basic workflow when authenticating to a service that uses SAML/OIDC from the users perspective is as follows:

1. User goes to the web address of a SSO protected service (known as a service provider, or SP).

2. The service asks for a cookie that the users browser may have stored, which contains a token, if it finds a valid token in the browser, it logs the user in. If this token does not exist, or is invalid, the service directs the user to the configured Identity Provider (IDp).

3. Once the user reaches the IDp, they are presented with a login page.

4. The user logins in with their provided credentials. The IDp can compare these to locally stored credentials, or against an LDAP or Kerberos backend.

5. The user is given a token upon successful login which is stored as a cookie in the browser, and gets automatically redirected back to the service they were initially attempting to access. This token usually contains a username, as well as information regarding what the user has access to, if using a protocol like OIDC that supports authorization.

6. The service requests the token from the browser and if the token is valid, allows the user access to the service.

Token validation is done on a secure back channel between the service and Identity Provider, without involving the browser, to increase security.
