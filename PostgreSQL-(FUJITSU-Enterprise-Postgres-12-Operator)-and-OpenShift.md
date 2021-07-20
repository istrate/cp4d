# PostgreSQL and OpenShift

https://www.postgresql.fastware.com/fujitsu-enterprise-postgres-for-kubernetes

This article contains several tips on how to deploy PostgreSQL in OpenShift cluster

# Install FEP Operator

The cluster administrator needs to deploy the FEP into the appropriate project (namespace). You can use Enterprise or Trial version.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-22%2021-16-47.png)

The trial version can be used only for testing and education and the license expires after 90 days.

# Install PostgreSQL instance

Installed Operator -> FUJITSU Enterprise Postgres 12 Operator -> FEPC FEPCluster -> Create Instance<br>

Change to YAML view and replace:
```
  max_worker_processes = 30
```
with 

```
  max_worker_processes = 42
```

Increase also the limit for CPU and memory, example:
```YAML
   mcSpec:
      limits:
        cpu: 2
        memory: 4G
      requests:
        cpu: 200m
        memory: 512Mi
```


Wait several minutes until the pod is ready.
# Test

Get *postgres* user password from *new-fep* secret (default is *admin-password*). The secret name corresponds to the name of FEP PostgreSQL created. <br>
Open pod terminal and run<br>
> psql -h localhost -p 27500 -U postgres
```
Password for user postgres: 
psql (12.5)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 postgres  | postgres | UTF8     | C       | C.UTF-8 | 
 template0 | postgres | UTF8     | C       | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C       | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(3 rows)

postgres=# 

```
# Recreate the instance

If you want to remove an existing cluster and deploy a new one, make sure that appropriate ConfigMaps are removed.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-22%2021-20-54.png)

Otherwise, the pod for the new FEP instance will run into a nasty CrashLoopBackOff event.

# Create an external access

As a default, the FEP operator creates only ClusterIP services. Create NodePort manually.<br>

> oc create service nodeport new-fep-sts --tcp=27500:27500

The service name (here *new-fep-sts*) should correspond to *app=new-fep-sts* label in the pod specification.

Make sure that service endpoint is defined.

> oc describe svc/new-fep-sts
```
name:                     new-fep-sts
Namespace:                postgresql
Labels:                   app=new-fep-sts
Annotations:              <none>
Selector:                 app=new-fep-sts
Type:                     NodePort
IP Families:              <none>
IP:                       172.30.121.200
IPs:                      172.30.121.200
Port:                     27500-27500  27500/TCP
TargetPort:               27500/TCP
NodePort:                 27500-27500  31505/TCP
Endpoints:                10.254.17.186:27500
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

```
The OpenShift assigned *31505* port number to the service.

On the HAProxy node, add a new role to HAProxy configuration. An example:
```
frontend ingress-fep
        bind *:27500
        default_backend ingress-fep
        mode tcp
        option tcplog

backend ingress-fep
        balance source
        mode tcp
        server master0 10.17.43.9:31505 check
        server master1 10.17.46.40:31505 check
        server master2 10.17.48.179:31505 check
        server worker0 10.17.57.166:31505 check
        server worker1 10.17.59.104:31505 check
        server worker2 10.17.61.175:31505 check
```

> systemctl restart haproxy<br>

Verify external access to the FEP instance.<br>
> psql -h \<HAProxy host\> -p 27500 -U postgres
```
Password for user postgres: 
psql (13.3, server 12.5)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 postgres  | postgres | UTF8     | C       | C.UTF-8 | 
 template0 | postgres | UTF8     | C       | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C       | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(3 rows)
```
