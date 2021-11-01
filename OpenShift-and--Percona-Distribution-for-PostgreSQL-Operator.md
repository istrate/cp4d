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

| Pod | Volume | Description
| --- | --- | --- | 
| cluster1- | Ephemeral | Main PostgreSQL instance. The data is stored in ephemeral storage but is replicated to *cluster1-repl* pods. In case of failure, the *cluster1-* pod can be deleted. The pod is next automatically recreated and the database is restored from *cluster1-repl* pod.
| cluster1-repl | PVC, default StorageClass | PostgreSQL replica. All changes in *cluster1-* instance is mirrored in replica.
| cluster1-backrest-shared-repo- | PVC, default StorageClass | Backup/Restore instance. Additional DR feature running *pgbackrest* utility. The backup schedule is defined in *PerconaPGClusters* instance.