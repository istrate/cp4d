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