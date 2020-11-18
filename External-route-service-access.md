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

## Discover ClusterIP

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

## Make ClusterIP accessible from Gateway node

CluserIP address *172.30.94.36* is not visible from Gateway node because it is the part of internal OpenShift network. 

On Gateway node<br>
<br>
Step 1: Create IP route to reach ClusterIP (172.30.94.36) assuming OpenShift Master node *10.16.35.203*
<br>
> ip route add   172.30.94.0/24 via 10.16.35.203 dev eth0<br>
<br>
Test *mysql-openshift* service (assuming MySQL client installed on Gateway node)

> mysql -uuser1 -pmypa55 --protocol tcp -h 172.30.94.36<br>
```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> 
```

Make IP route permanent to survive network restart.<br>

> vi /etc/sysconfig/network-scripts/route-eth0<br>
```
ADDRESS0=172.30.94.0
NETMASK0=255.255.255.0
GATEWAY0=10.16.35.203
```

# Configure HAProxt on Gateway node

Assuming public IP address of Gateway node is *9.30.220.176*. Forward all tradif on *3306* port to ClusterIP.
<br>

> vi /etc/haproxy/haproxy.cfg 
```
listen mysql
        bind 9.30.220.176:3306
        mode tcp
        server server1 172.30.94.36:3306 check

```

Restart HAProxy.

> systemctl restart haproxy

## Test from the client desktop
<br>
> mysql -uuser1 -pmypa55 --protocol tcp -h mysql-openshift-sb.apps.rumen.os.fyre.ibm.com<br>

```
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 5.7.24 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```
