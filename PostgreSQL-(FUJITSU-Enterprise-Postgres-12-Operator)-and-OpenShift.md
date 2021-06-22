# PostgreSQL and OpenShift

https://www.postgresql.fastware.com/fujitsu-enterprise-postgres-for-kubernetes

This article contains several tips on how to deploy PostgreSQL in OpenShift cluster

# Install FEP Operator

The cluster administrator needs to deploy the FEP into the appropriate project (namespace). You can use Enterprise or Trial version.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-22%2021-16-47.png)

The trial version can be used only for testing and education and the license expires after 90 days.

# Install PostgreSQL instance

Installed Operator -> FUJITSU Enterprise Postgres 12 Operator -> FEPC FEPCluster -> Create Instance<br>

Change ti YAML view and replace:
```
  max_worker_processes = 30
```
with 

```
  max_worker_processes = 42
```

and wait several minutes until the pod is ready.

# Test

Get *postgres* user password from *new-fep* secret (default is *admin-password*). The secret name corresponds to the name of FEP PostgreSQL created. <br>
Open pod terminal and run<br>
> psql -h localhost -p 27500 -u postgres
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


