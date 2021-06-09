# Practical steps on how to deploy MariaDB to OpenShift cluster

## Docker

Make sure you have access to *registry.redhat.io* registry. <br>

Simple test, if fails try another docker registry.<br>

> podman pull registry.redhat.io/rhscl/mariadb-105-rhel7
```
Trying to pull registry.redhat.io/rhscl/mariadb-105-rhel7:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 9b0c218cbfb1 done  
Copying blob f8c518873786 done  
Copying blob 93156a512b98 [=>------------------------------------] 3.6MiB / 72.9MiB
Copying blob d133eb51475d [--------------------------------------] 711.1KiB / 60.6MiB
```
## Create user and project

As OpenShift *admin* user, create *mariadb* admin user.<br>

> htpasswd -nb mariadb secret<br>

Modify *OAuth* secret and verify *mariadb* credentials.<br>

> oc login -u mariadb -p secret<br>

As *admin* user.

> oc new-project mariadb<br>
> oc adm policy add-role-to-user admin mariadb -n mariadb<br>

## Create MariaDB instance

As *mariadb* user in *mariadb* project.<br>

> oc login -u mariadb -p secret<br>

> oc new-app --docker-image=registry.redhat.io/rhscl/mariadb-105-rhel7  --name=mariadb  -e MYSQL_ROOT_PASSWORD=secret<br>

Assign Persistent Volume.<br>

> oc set volume deployment/mariadb --add  --claim-name=mariadb  --mount-path=/var/lib/mysql/data  -t pvc --claim-size=10G  

> oc expose deployment/mariadb --name mariadbn --type=NodePort

## Verify the deployment

> oc get pods
```
NAME                       READY   STATUS    RESTARTS   AGE
mariadb-547c6d45fd-4pfbz   1/1     Running   0          49m
```
> oc rsh mariadb-547c6d45fd-4pfbz<br>

> mysql -u root
```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 21
Server version: 10.5.9-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```
## Allow external access

Assuming HAProxy node *kist* between OpenShift cluster and external network.<br>

> oc expose deployment/mariadb --name mariadbn --type=NodePort<br>

> oc get svc
```
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mariadb    ClusterIP   172.30.96.56     <none>        3306/TCP         69m
mariadbn   NodePort    172.30.161.143   <none>        3306:32736/TCP   30m
```

The external port is *32736*. 

Modify HAProxy configuration on infrastructure node. It is an example, adjust IP addresses according to your cluster configuration.<br>
> vi /etc/haproxy/haproxy.cfg<br>
```
frontend ingress-mariadb
        bind *:3306
        default_backend ingress-mariadb
        mode tcp
        option tcplog

backend ingress-mariadb
        balance source
        mode tcp
        server master0 10.17.43.9:32736 check
        server master1 10.17.46.40:32736 check
        server master2 10.17.48.179:32736 check
        server worker0 10.17.57.166:32736 check
        server worker1 10.17.59.104:32736 check
        server worker2 10.17.61.175:32736 check
```
> systemctl restart haproxy

Verify port on *kist* node.<br>
> nc -zv kist 3306
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.46.197.101:3306.
Ncat: 0 bytes sent, 0 bytes received in 0.24 seconds.
```

Try to connect to MariaDB instance.<br>

>mysql -h kist -u root -p
```
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 22
Server version: 5.5.5-10.5.9-MariaDB MariaDB Server

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```


