# PostgreSQL and OpenShift

https://www.postgresql.fastware.com/fujitsu-enterprise-postgres-for-kubernetes

This article contains several tips on how to deploy PostgreSQL in OpenShift cluster

# Install FEP Operator

The cluster administrator needs to deploy the FEP into the appropriate project (namespace). You can use Enterprise or Trial version.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-22%2021-16-47.png)

Trial version can be used only for testing and education and the license expires after 90 days.

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