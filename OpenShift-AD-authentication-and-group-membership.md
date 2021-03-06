# OpenShift and AD

OpenShift AD/LDAP integration is a more flexible authentication and authorization method compared to the HTPasswd identity provider. It involves two steps: authentication and authorization, group membership.<br>

This article covers only a basic scenario. There are a lot more options covered in OpenShift 4.7 documentation.

# Authentication

## Collect data

https://docs.openshift.com/container-platform/4.7/authentication/identity_providers/configuring-ldap-identity-provider.html

Before configuring, collect all necessary information. Let's describe my AD configuration.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-15%2013-38-22.png)

| Description | Description | Value |
| ---------- | ---------- | ----- |
| AD hostname | | verse1.fyre.ibm.com
| Port number | Usually 389 for non-secure and 636 for secure connection | 389
| Secure/non-secure | Secure connection requires certificate | false
| baseDN | Starting point in AD tree to look for the users | cn=centos,DC=fyre,DC=net
| binding user | Read-only user authorized to scan AD directory tree. Required when AD does not allow anonymous access | hadoopsearch
| binding user password | AD password for binding user | secret
| id | Attribute used for identity | sAMAccountName

# Create a secret with a binding password

> oc create secret generic ldap-secret --from-literal=bindPassword=secret -n openshift-config

# Configure LDAP identity

> oc edit OAuth<br>
```YAML
spec:
  identityProviders:
  - name: ldapidp 
    mappingMethod: claim 
    type: LDAP
    ldap:
      attributes:
        id: 
        - sAMAccountName
        email: 
        - mail
        name: 
        - name
        preferredUsername: 
        - sAMAccountName
      bindDN: hadoopsearch 
      bindPassword: 
        name: ldap-secret
      insecure: true
      url: "ldap://verse1.fyre.ibm.com:389/cn=centos,DC=fyre,DC=net?sAMAccountName" 
```
# Authorization, group membership

https://docs.openshift.com/container-platform/4.7/authentication/ldap-syncing.html

AD - OpenShift group synchronization is a separate activity. It is done on-demand, every change in AD group is reflected in OpenShift only after running the synchronization job.<br>

Prepare synchronization config file.<br>
> vi config.yaml
```
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://verse1.fyre.ibm.com:389
insecure: true
bindDN: hadoopsearch
bindPassword: secret
augmentedActiveDirectory:
    groupsQuery:
        baseDN: "cn=centos,DC=fyre,DC=net"
        scope: sub
        derefAliases: never
        pageSize: 0
    groupUIDAttribute: dn 
    groupNameAttributes: [ cn ] 
    usersQuery:
        baseDN: "cn=centos,dc=fyre,dc=net"
        scope: sub
        derefAliases: never
        filter: (objectclass=person)
        pageSize: 0
    userNameAttributes: [ sAMAccountName ] 
    groupMembershipAttributes: [ memberOf ] 
```

>  oc adm groups sync --sync-config=config.yaml --confirm<br>

It *--confirm* flags is removed, it is a dry run, the command reports the changes to be applied without executing them.<br>

The command is dealing with groups and group membership but does not create or remove any users. If in the command output a user is reported as belonging to the group, it means only that user name is bound to the group but in order to make it effective, the user identity should be created in OpenShift.<br>

The command does not remove groups. If the group has been removed in AD and is expected to be removed also in OpenShift, a separate job is needed.

>  oc adm prune groups --sync-config=config.yaml --confirm<br>

