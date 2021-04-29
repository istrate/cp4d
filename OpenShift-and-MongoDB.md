# MongoDB and Percona Operator

https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html

It is very easy to set up MongoDB cluster using Percona Server MongoDB Operator.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2011-59-46.png)

# Install Percona Operator

As OpenShift cluster-admin, create a project, project admin and deploy the operator.

> oc new-project mongodb<br>
> htpasswd -nb mongoadmin secret
```
mongoadmin:$apr1$7eJnodEE$1HErN.W2Lweq6.BU4VjR5.
```
Update OAUth entry and make sure that *mongoadmin* user can log on.<br>

> oc adm policy add-role-to-user admin mongoadmin  -n mongodb<br>

From OpenShift Console install Percona Operator into *mongodb* project.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2012-49-06.png)

# Install MongoDB Server

As *mongoadmin* user, install MongoDB ServerDB cluster. A simple cluster: single shard and three replicas, accepts defaults without changing anything.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-19-40.png)

Wait until pods are ready.<br>

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-27-40.png)

# Get credentials

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-31-38.png)

"Action" -> "Edit Secret" and retrieve "userAdmin" password.

# Verify connectivity






