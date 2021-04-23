# DB2 and OpenShift

Containerized DB2 is available for RedHat OpenShift. The installation is easy and straightforward also in a tiny environment.

# Enable DB2 Operator

Follow the instruction.<br>

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/doc/t_db2u_install_op_catalog.html

# Install DB2 Operator

Create a separate OpenShift project to maintain DB2 instances.<br>

> oc new-project db2<br>

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

# Storage options

https://www.ibm.com/support/producthub/db2/docs/content/SSEPGG_11.5.0/com.ibm.db2.luw.db2u_openshift.doc/database-storage-aese.html

There are several options available. For testing and evaluating, NFS storage is a good option. 

https://github.com/stanislawbartkowski/CP4D/wiki/OpenShift-NFS-provisioner

OpenShift Container Storage. https://www.openshift.com/blog/introducing-openshift-container-storage-4-2
<br>
Rook-Ceph: use *rook-cephfs* (file system)

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

Identify the Service Account, the name can be different according to the release name. Here: *dbu-release*.

>oc get a<br>
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
