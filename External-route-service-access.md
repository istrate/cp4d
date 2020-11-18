# OpenShift routes

https://docs.openshift.com/container-platform/4.5/networking/routes/route-configuration.html

OpenShift route is an extension to Kubernetes service allowing external access to OpenShift/Kubernetes applications. But it requires at least one of the OpenShift nodes to be accessible directly from the client node.<br>

# Architecture

Assume the architecture where the whole OpenShift cluster is running on a private network and the gateway is a separate node running in both networks, private and public, but not being included in the OpenShift cluster.

| Node | Network | Role 
| --- | ---- | ---- 
| master0.oc.com | private | Master
| worker0.oc.com | private | Worker
| worker1.oc.com | private | Worker
| worker2.oc.com | private | Worker
| inf.oc.com | private, public | Gateway to the cluster, HAProxy, NFS services
| client.sb.com | public | Client desktop

The client can access the gateway node and run OpenShift console or *oc* command utilizing the HAProxy gateway node service.

# Create MySQL instance

<br>

> oc new-app --as-deployment-config  --docker-image=registry.access.redhat.com/rhscl/mysql-57-rhel7:latest --name=mysql-openshift  -e MYSQL_USER=user1 -e MYSQL_PASSWORD=mypa55 -e MYSQL_DATABASE=testdb  -e MYSQL_ROOT_PASSWORD=r00tpa55<br>

<br>
This command creates MySQL instance and MySQL service.
<br>

> oc get svc<br>

```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql-openshift   ClusterIP   172.30.94.36   <none>        3306/TCP   4d12h
```

<br>

Make sure that MySQL database is up and running.<br>
<br>

>oc get pods<br>
<br>

```
NAME                      READY   STATUS    RESTARTS   AGE
mysql-openshift-1-85ft6   1/1     Running   1          26h
```


> oc port-forward mysql-openshift-1-85ft6  3306:3306<br>
```
Forwarding from 127.0.0.1:3306 -> 3306
Forwarding from [::1]:3306 -> 3306

```
Using another command-line console <br>

> mysql -uuser1 -pmypa55 --protocol tcp -h 127.0.0.1<br>
```
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

# Create a route

> oc expose service mysql-openshift<br>
> oc get route<br>
```
NAME              HOST/PORT                                       PATH   SERVICES          PORT       TERMINATION   WILDCARD
mysql-openshift   mysql-openshift-sb.apps.rumen.os.fyre.ibm.com          mysql-openshift   3306-tcp                 None

```
Route exposes *mysql-openshift-sb.apps.rumen.os.fyre.ibm.com* hostname and *3306* port. But hostname points to Gateway which is not part of the OpenShift cluster and Gateway is unable to forward the port to OpenShift route.

> oc delete route mysql-openshift

# Solution using the service

> oc describe service mysql-openshift<br>

```
Name:              mysql-openshift
Namespace:         sb
Labels:            app=mysql-openshift
                   app.kubernetes.io/component=mysql-openshift
                   app.kubernetes.io/instance=mysql-openshift
.....
Type:              ClusterIP
IP:                172.30.94.36
Port:              3306-tcp  3306/TCP
TargetPort:        3306/TCP
Endpoints:         10.254.12.23:3306
Session Affinity:  None
```

CluserIP address *172.30.94.36* is not visible from gateway node because it is the part of internal OpenShift network. 