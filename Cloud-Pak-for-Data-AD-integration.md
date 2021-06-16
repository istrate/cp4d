# CP4D and LDAP/AD authentication

https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=users-connecting-your-ldap-server

This article describes how to integrate CP4D with AD server specified under this link: https://github.com/stanislawbartkowski/CP4D/wiki/OpenShift-AD-authentication-and-group-membership

# Collect necessary data

| Field | Value
| ---- | ----
| LDAP protocol | ldap:// (non secure connection)
| LDAP hostname | verse1.fyre.ibm.com
| LDAP port | 389 (default non-secure)
| User search base | cn=centos,DC=fyre,DC=net
| User search field | sAMAccountName
| Domain search user | Only if AD/LDAP requires authentication, here: hadoopsearch
| Domain user password | secret
| Group search base | cn=centos,DC=fyre,DC=net
| Group search field | cn
| First name | givenName
| Last name | sn
| Email | mail
| Group membership | memberOf
| Group member field | member

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-16%2012-31-07.png)

