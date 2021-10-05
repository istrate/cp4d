# DB2 and OpenShift

Containerized DB2 is available for RedHat OpenShift. The installation is easy and straightforward also in a tiny environment.

# Enable DB2 Operator

Follow the instruction.<br>

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/doc/t_db2u_install_op_catalog.html

# Important

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-15%2001-39-58.png)

# Install DB2 Operator

Create a separate OpenShift project to maintain DB2 instances.<br>

> oc new-project db2<br>

Create also additional *db2admin* user to install and manage DB2 cluster. After the operator is installed, *db2admin* will provision and manage DB2 cluster. When *db2admin* is created, assign *admin* role in *db2* project.

> oc adm policy add-role-to-user admin db2admin -n db2<br>

In OpenShift console, go to Operators->OperatorHub and search for "Db2". Click "IBM Db2" tile and wait a moment until the info page is displayed. Deploy *Entitlement Key* following the instruction attached to the operator. Make sure to pass all points: 1,2 and 3.
```
............
To install the Db2U Operator using the command-line
1. Retrieve an entitled key from the Entitled registry
Log into MyIBM
Copy the Entitled Key
............
```

Click "Install" and select "db2" namespace just created as a placeholder for Db2.<br>
<br>
Deploy *Entitlement Key* following an instruction attached to the operator. In *db2* project, go to "Installed Operators" and click the line. After a while, the web page is displayed. 
```
............
To install the Db2U Operator using the command-line
1. Retrieve an entitled key from the Entitled registry
Log into MyIBM
Copy the Entitled Key
............
```
# Pull secret

Make sure that *pull secrets* are present in the DB2Cluster *yaml*.

```
  account:
    privileged: true
    imagePullSecrets:
      - ibm-registry  
```

Another method is to make *ibm-registry* available across the OCP cluster as described in DB2 Operator help text - chapter 2.


# Storage options

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/database-storage-aese.html

There are several options available. For testing and evaluating, NFS storage is a good option. 

https://github.com/stanislawbartkowski/CP4D/wiki/OpenShift-NFS-provisioner

OpenShift Container Storage. https://www.openshift.com/blog/introducing-openshift-container-storage-4-2
<br>
Rook-Ceph: use *rook-cephfs* (file system)

More detailed storage assignment.<br>
* data: root-ceph-block
* meta data: rook-cephfs
* backup: managed-nfs-storage

Important: make sure that in the YAML file, the *storage* key is at the same level as the *version*. Otherwise, no error is reported but DB2 cluster will not be created.

```
  version: 11.5.5.0-cn4

  storage:
    - name: meta
      type: create
      spec:
        storageClassName: rook-cephfs
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 10Gi
    - name: data
      type: create
      spec:
        storageClassName: rook-ceph-block
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
    - name: backup
      type: create
      spec:
        storageClassName: managed-nfs-storage
        accessModes:
         - ReadWriteMany
        resources:
          requests:
            storage: 50Gi

```


# Install DB2 database

The simplest option is "Db2u Cluster". "OC Console" -> Project db2 -> Installed Operators -> IBM DB2 -> Db2u Cluster -> Create Instance<br>

In order to use non-default storage, open "YAML View" and enter appropriate StorageClass name in the *storageClassName* property.<br>

It can take half an hour to provision DB2 instance for NFS storage and around 10 minutes in case of Ceph backed storage.<be>

# ImagePullBackOff error

Verify that credential are valid. As a password, paste *Entitlement Key* <br>
<br>
> podman login  cp.icr.io -u cp
```
Password: 
Login Succeeded!
```
Identify the URL image from *pod* event and pull it manually as a test.
<br>
> podman pull cp.icr.io/cp/db2u.instdb@sha256:2f7d28e01153aaf516e750ba6463fb21d867cba2c66fa982b64f20af01e1434a
```
Trying to pull cp.icr.io/cp/db2u.instdb@sha256:2f7d28e01153aaf516e750ba6463fb21d867cba2c66fa982b64f20af01e1434a...
Getting image source signatures
Copying blob 4a876aa7c77a done  
Copying blob 2814fe1cf24f done  
Copying blob af076c7c771c done  
```

Verify that *Entitlement Key* is pasted correctly in *ibm-registry*.

Identify the Service Account, the name can be different according to the release name. Here: *db2u-release*.

>oc get sa<br>
```
NAME                       SECRETS   AGE
account-db2-db2u-release   2         38m
builder                    2         8d
...
```
Assign DB2 Service Account to DB2 pull secret.

> oc secrets link account-db2-db2u-release ibm-registry --for=pull<br>

Delete failed pods and wait until recreated.

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
# Open external access to OpenShift DB2

DB2 operator creates a number of services related to DB2. One method to use OpenShift *route* netowrking but this option is not always available.<br>
Another method is to redirect ports in *HAProxy* gateway on the infrastructure node.<br>

Get service ports.<br>

> oc project db2<br>
> oc get svc<br>
```
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                           AGE
c-db2ucluster-sample-db2u            ClusterIP   172.30.83.77     <none>        50000/TCP,50001/TCP,25000/TCP,25001/TCP,25002/TCP,25003/TCP,25004/TCP,25005/TCP   12m
c-db2ucluster-sample-db2u-engn-svc   NodePort    172.30.155.188   <none>        50000:31753/TCP,50001:30397/TCP                                                   12m
c-db2ucluster-sample-db2u-internal   ClusterIP   None             <none>        50000/TCP,9443/TCP                                                                12m
c-db2ucluster-sample-etcd            ClusterIP   None             <none>        2379/TCP,2380/TCP                                                                 13m
c-db2ucluster-sample-ldap            ClusterIP   172.30.59.53     <none>        50389/TCP                                                                         14m
c-db2ucluster-sample-tools           ClusterIP   172.30.32.134    <none>        53/TCP,53/UDP        
```
For *c-db2ucluster-sample-db2u-engn-svc* pod take notice of *NodePort*, *31753* port for non-secure connection and *30397* port for secure connection.<br>

Get *ip* addresses for master node(s).<br>
```
NAME                               STATUS   ROLES    AGE   VERSION
master0.bewigged.os.fyre.ibm.com   Ready    master   24d   v1.19.0+7070803
master1.bewigged.os.fyre.ibm.com   Ready    master   24d   v1.19.0+7070803
master2.bewigged.os.fyre.ibm.com   Ready    master   24d   v1.19.0+7070803
worker0.bewigged.os.fyre.ibm.com   Ready    worker   24d   v1.19.0+7070803
worker1.bewigged.os.fyre.ibm.com   Ready    worker   24d   v1.19.0+7070803
worker2.bewigged.os.fyre.ibm.com   Ready    worker   24d   v1.19.0+7070803
worker3.bewigged.os.fyre.ibm.com   Ready    worker   24d   v1.19.0+7070803
worker4.bewigged.os.fyre.ibm.com   Ready    worker   24d   v1.19.0+7070803

```
Modify *HAProxy* configuration. Replace hostname with corresponding *ip* addresses and restart *haproxy* service.

> vi /etc/haproxy/haproxy.cfg<br>
```
.........
frontend db2
        bind *:31753
        default_backend db2u
        mode tcp
        option tcplog

backend db2u
        balance source
        mode tcp
        server master0 10.16.71.16:31753 check
        server master1 10.16.71.50:31753 check
        server master2 10.16.71.52:31753 check


frontend db2ssl
        bind *:30397
        default_backend db2ussl
        mode tcp
        option tcplog

backend db2ussl
        balance source
        mode tcp
        server master0 10.16.71.16:30397 check
        server master1 10.16.71.50:30397 check
        server master2 10.16.71.52:30397 check

```
# Verify using *DB2* CLP

Password for *db2inst1* user is generated randomly and stored in related secret. In *Secret* panel, identify the line similar to *c-db2ucluster-sample-instancepassword* and copy the content. Example: XeYnwqqWcrbfiLx

Assuming hostname of infrastructure node *bewigged-inf*.<br>

> db2 catalog tcpip node DB2BLU remote bewigged-inf server 31753<br>
> db2 catalog database BLUDB at node DB2BLU<br>

Try to connect using *db2inst1* and password.<br>

> db2 attach to db2blu user db2inst1<br>
```
Enter current password for db2inst1: 

   Instance Attachment Information

 Instance server        = DB2/LINUXX8664 11.5.5.0
 Authorization ID       = DB2INST1
 Local instance alias   = DB2BLU
```
# Verify using db2cli

>  db2cli  execsql  -connstring "DATABASE=BLUDB;HOSTNAME=adown-inf;PORT=50000;UID=db2inst1;PWD=1XmevGtqeTa8lxe"<br>
```
IBM DATABASE 2 Interactive CLI Sample Program
(C) COPYRIGHT International Business Machines Corp. 1993,1996
All Rights Reserved
Licensed Materials - Property of IBM
US Government Users Restricted Rights - Use, duplication or
disclosure restricted by GSA ADP Schedule Contract with IBM Corp.
> 
```
# Connection on a secure port using DBeaver

A secure connection can be exposed as OpenShift route.

> oc get scv<br>
```
c-db2ucluster-db2u-engn-svc   NodePort    172.30.138.45    <none>        50000:31847/TCP,50001:31883/TCP   
```
> oc create route passthrough db2route --service=c-db2ucluster-db2u-engn-svc --port=50001<br>
> oc get route<br>
```
NAME       HOST/PORT                                 PATH   SERVICES                      PORT    TERMINATION   WILDCARD
db2route   db2route-db2.apps.adown.cp.fyre.ibm.com          c-db2ucluster-db2u-engn-svc   50001   passthrough   None
```
The connection hostname is *db2route-db2.apps.adown.cp.fyre.ibm.com* and the port is *443* (not 50001). Secure port *443* is redirected by internal OpenShift LoadBalancer to 50001 secure port in DB2 instance.<br>

Verify SSL connection.<br>
> openssl s_client -connect db2route-db2.apps.adown.cp.fyre.ibm.com:443<br>
```
.............
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : AES256-GCM-SHA384
    Session-ID: 64B271D48EE8D18DEE2CD7C7D2E4098AB346AA1317A9AB4494C4E2087F4019EF
    Session-ID-ctx: 
    Master-Key: 73AA0534922AEAEEC02B4189C7FD83D5E3C9A40DDC170E57D0C47C1292687F7A83CF60CB6823D91C6D504D8F733F1656
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1621025764
    Timeout   : 7200 (sec)
    Verify return code: 19 (self signed certificate in certificate chain)
    Extended master secret: yes
---
```

Copy and paste server certificate from *openssl* output.
```
-----BEGIN CERTIFICATE-----
MIIEgzCCAmugAwIBAgIJAOWcmi2nUUX2MA0GCSqGSIb3DQEBCwUAMF8xCzAJBgNV
......................
+qBKrHnk1A==
-----END CERTIFICATE-----

```
Save the certificate in */tmp/db2.crt* file and create a keystore protected by a password.
> keytool -import -file /tmp/db2.crt -keystore {directory}/server.jks<br>

In DBeaver connection wizard, add the following parameters to database name. Pay attention to colon (:) at the beginning and semicolon(;) at the end. *:sslConnection=true;sslTrustStoreLocation={directory}/server.jks;sslTrustStorePassword=secret;*

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-05-14%2023-03-45.png)


# Backup and restore

Backup and restore in DB2 Warehouse is controlled by the same command as standalone DB2 Instance. The default backup directory is */mnt/backup*. Make sure that the backup directory is externalized and attached to DB2 pod as PV. <br>
Identify the pod running DB2 Warehouse instance.<br>

> oc get pod<br>
```
NAME                                         READY   STATUS      RESTARTS   AGE
c-db2ucluster-sample-db2u-0                  1/1     Running     0          3h25m
c-db2ucluster-sample-etcd-0                  1/1     Running     0          3h25m
c-db2ucluster-sample-instdb-qfpwl            0/1     Completed   0          3h25m
c-db2ucluster-sample-instdb-xxg9s            0/1     OOMKilled   0          3h26m
c-db2ucluster-sample-ldap-69777ffc89-9b2gc   1/1     Running     0          3h26m
c-db2ucluster-sample-restore-morph-777p2     0/1     Completed   0          3h17m
c-db2ucluster-sample-tools-b79bdfd67-rdl86   1/1     Running     0          3h26m
db2u-operator-manager-d6cf846f4-c6brl        1/1     Running     0          3h37m
```
> oc rsh c-db2ucluster-sample-db2u-0<br>

> su - db2inst1<br>
> db2 connect to bludb<br>
> db2 quiesce database immediate force connections <br>
> db2 deactivate db bludb<br>
> db2 backup database bludb to /mnt/backup
```
Backup successful. The timestamp for this backup image is : 20210510155003
```
Off-line backup is done. Not bring database back to live again<br>
> db2 connect to bludb<br>
> db2 unquiesce database<br>
<br>
Now find the backup file and move the file to another instance where the database is going to be restored. On the source DB2, the */mnt/backup* is attached as NFS Storage Class.
<br>

> ls /data/db2-c-db2ucluster-sample-backup-pvc-191bb80e-446d-4c5f-b2a4-89cae6f3ad6f/ -ltr
```
May 10 08:50 BLUDB.0.db2inst1.DBPART000.20210510155003.001
```
Copy the file to the */mnt/backup* PV of target DB2 Warehouse instance.<br>

> scp /data/db2-c-db2ucluster-sample-backup-pvc-191bb80e-446d-4c5f-b2a4-89cae6f3ad6f/BLUDB.0.db2inst1.DBPART000.20210510155003.001 root@9.46.102.217:/data/nfs/db2-c-db2ucluster-sample-backup-pvc-b5a51060-0b50-4806-9c0a-597570a6c1bd/

On the target DB2 Warehouse, identity the DB2 Warehouse pod and enter the pod.<br>
> oc rsh c-db2ucluster-sample-db2u-0 <br>
> su - db2inst1<br>
> ls /mnt/backup
```
BLUDB.0.db2inst1.DBPART000.20210510155003.001
```
> db2 connect to bludb<br>
> db2 quiesce database immediate <br>
> db2 terminate<br>
> db2 deactivate db bludb<br>
> db2 restore db bludb from /mnt/backup taken at 20210510155003<br>
```
SQL2523W  Warning!  Restoring to an existing database that is different from 
the database on the backup image, but have matching names. The target database 
will be overwritten by the backup version.  The Roll-forward recovery logs 
associated with the target database will be deleted.
Do you want to continue ? (y/n) y

```
> db2 rollforward db bludb complete<br>
```
db2 rollforward db bludb complete

                                 Rollforward Status

 Input database alias                   = bludb
 Number of members have returned status = 1

 Member ID                              = 0
 Rollforward status                     = not pending
 Next log file to be read               =
 Log files processed                    =  -
 Last committed transaction             = 2021-05-10-15.50.41.000000 UTC

DB20000I  The ROLLFORWARD command completed successfully.

```
Several useful commands.<br>

Release instance memory<br>

> db2 update dbm cfg using INSTANCE_MEMORY AUTOMATIC<br>

Enable column-oriented table<br>

> db2 update dbm cfg using INTRA_PARALLEL YES<br>

# SQL1564N during RESTORE
```
SQL1564N  The restore or rollforward operation did not complete successfully 
because the specified operation is not supported. Reason code "9".

```
Add *WITHOUT ROLLING FORWARD*

> db2 restore db bludb from  /mnt/backup/backup_off_1  taken at 20210425102241 WITHOUT ROLLING FORWARD






