# DB2 and OpenShift

Containerized DB2 is available for RedHat OpenShift. The installation is easy and straightforward also in a tiny environment.

# Enable DB2 Operator

Follow the instruction.<br>

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/doc/t_db2u_install_op_catalog.html

# Install DB2 Operator

Create a separate OpenShift project to maintain DB2 instances.<br>

> oc new-project db2<br>

In OpenShift console, go to Operators->OperatorHub and search "Db2". Click "Install" and select "db2" namespace just created as a placeholder for Db2.<br>
<br>
Deploy *Entitlement Key* following an instruction attached to the operator. In *db2* project, go to "Installed Operators" and click the line. After a while, the web page is displayed. 

# Storage options

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/database-storage-aese.html

There is a number of options available. For testing and evaluating, NFS storage is possible. 

https://github.com/stanislawbartkowski/CP4D/wiki/OpenShift-NFS-provisioner

Another option is OpenShift Container Storage. https://www.openshift.com/blog/introducing-openshift-container-storage-4-2

# Install DB2 database

The simplest option is "Db2u Cluster". "OC Console" -> Project db2 -> Installed Operators -> IBM DB2 -> Db2u Cluster -> Create Instance<br>

In order to use non-default storage, open "YAML View" and enter appropriate StorageClass name in the *storageClassName* property.

# Verify that DB2 is up and running

Go to pod related to *db2ucluster* and open terminal<br>
> su - db2inst1<br>
> db2 list db directory<br>
> db2 connect to BLUDB<br>
```
   Database Connection Information

 Database server        = DB2/LINUXX8664 11.5.5.0
 SQL authorization ID   = DB2INST1
 Local database alias   = BLUDB
```






