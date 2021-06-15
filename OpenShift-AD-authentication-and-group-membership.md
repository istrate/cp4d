# OpenShift and AD

OpenShift AD/LDAP integration is a more flexible authentication and authorization method compared to the HTPasswd identity provider. It involves two steps: authentication and authorization, group membership.

# Authentication

https://docs.openshift.com/container-platform/4.7/authentication/identity_providers/configuring-ldap-identity-provider.html

Before configuring, collect all necessary information. Let's describe my AD configuration.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-15%2013-38-22.png)

| Description | Description | Value |
| ---------- | ---------- | ----- |
| AD hostname | | verse1.fyre.ibm.com
| Port number | Usually 389 for non-secure and 636 for secure connection | 389
| Secure/non-secure 