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

Cloud Pak for Data -> Administration -> Access control -> LDAP configuration <br>


![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-16%2012-31-07.png)

At the bottom, there is an additional section to test the connection. Enter the credential of any user in the LDAP server and click *Test Connection*. This credential is used only for the test and is not stored.

# New user

Adding a new user from LDAP pool can be tricky. Firstly log on as a new user and this first trial will be rejected. But new user entry is added to CP4D user list. The next step is to log on as the *admin* user and assign the new user any role, for instance, *user* role. Then the next log on will be successful.