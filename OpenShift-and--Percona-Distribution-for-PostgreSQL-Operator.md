# Introduction

https://github.com/percona/percona-postgresql-operator

Practical information on setting up PostgreSQL instance on OpenShift.

# Installation

## Project and user

Create a separate project for PostgreSQL instance, for instance, *postgres*. The installation and management can be done by *cluster-admin* user or by a separate user having admin authority only in *postgres* project.

## Deploy the operator

An operator can be deployed only by *cluster-admin*.<br>Browse to OpenShift console *Operator Hub* panel.<br>
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-11-01%2012-25-10.png)

Install the operator to *postgres* project.

## Create PostgreSQL instance

It can be done by a user with *admin* authority in *postgres* project.<br>
The installation is very simple and straightforward.<br>

OpenShift console -> *postgres* project -> Installed Operators -> Percona Distribution for PostgreSQL Operator<br>

Click *Create PerconaPGCluster* and *Create* button. Wait until all pods are created and running.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-11-01%2012-33-46.png)

## Test

Open "Terminal" in *cluster1-* pod, run *psql* command and execute several commands.<br>

> psql
```
psql (13.4)
Type "help" for help.

postgres=# 
```
> create database testdb;<br>
> \c testdb;<br>
> create table test x (int);<br>
> insert into test values (0),(1),(2);<br>
> select * from x;
```
x 
---
 0
 1
 2
(3 rows)
```
## Pods description

| Pod | Description
| --- | --- | 
| cluster1-pgbouncer | Connection pooler and load balancer for PosgreSQL cluster. 
| cluster1-<br>cluster1-repl | Three PostgreSQL instances running in Master/StandBy mode. One is active and two are standbys. Every transaction committed on the active instance is replicated on standbys. The application should avoid connecting to any of the containers directly because standby instances are running in read-only mode. Connection to *pgbouncer* is recommended.
| cluster1-backrest-shared-repo- | Backup/Restore instance. Additional DR feature running *pgbackrest* utility. The backup schedule is defined in *PerconaPGClusters* instance.

## External access

### Create NodePort

Replace *selector* with valid pod label (here *name=cluster1-pgbouncer*). The selector should point to *pgbouncer* container.<br>
```
cat <<EOF |oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgresql
spec:
  selector:           
    name: cluster1-pgbouncer
  type: NodePort
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432         
EOF
```

Check that the service selector is resolved into valid pod IP, *endpoint* contains *cluster1-* pod IP.

> oc describe svc/postgresql<br>
```
Name:                     postgresql
Namespace:                postgresql
Labels:                   <none>
Annotations:              <none>
Selector:                 name=cluster1
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       172.30.196.101
IPs:                      172.30.196.101
Port:                     <unset>  5432/TCP
TargetPort:               5432/TCP
NodePort:                 <unset>  32026/TCP
Endpoints:                10.254.20.71:5432
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

The external NodePort is assigned as *32076*.
<br>
### Configure haproxy

> vi /etc/haproxy/haproxy.cfg<br>
```
frontend ingress-postgresql
        bind *:5432
        default_backend ingress-postgresql
        mode tcp
        option tcplog

backend ingress-postgresql
        balance source
        mode tcp
        server master0 10.17.43.9:32026 check
        server master1 10.17.46.40:32026 check
        server master2 10.17.48.179:32026 check
        server worker0 10.17.57.166:32026 check
        server worker1 10.17.59.104:32026 check
        server worker2 10.17.61.175:32026 check

```

> systemctl restart haproxy<br>

### Test access

Assuming that *kist* is the hostname of *haproxy* node.<br>

> nc -zv kist 5432
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.46.197.101:5432.
Ncat: 0 bytes sent, 0 bytes received in 0.26 seconds.
```

### Connect externally to PostgreSQL

Get password for *postgres* admin user.<br>

> oc extract secret/cluster1-postgres-secret  --keys=password --to=-
```
# password
gHUksg0rPQJlqDFQFvuqxppC
```

Connect as *postgres* user. Important: while connecting to *pgbouncer*, the database name should be specified and *-W* option to force password.<br>

> psql -h kist -U postgres
```
Password for user postgres: 
psql (13.4)
Type "help" for help.

postgres=# 
```