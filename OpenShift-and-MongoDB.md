# MongoDB and Percona Operator

https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html

It is very easy to set up MongoDB cluster using Percona Server MongoDB Operator.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2011-59-46.png)

# Install Percona Operator

As OpenShift cluster-admin, create a project, project admin and deploy the operator.

> oc new-project mongodb<br>
> htpasswd -nb mongoadmin secret
```
mongoadmin:$apr1$7eJnodEE$1HErN.W2Lweq6.BU4VjR5.
```
Update OAUth entry and make sure that *mongoadmin* user can log on.<br>

> oc adm policy add-role-to-user admin mongoadmin  -n mongodb<br>

From OpenShift Console install Percona Operator into *mongodb* project.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2012-49-06.png)

# Install MongoDB Server

As *mongoadmin* user, install MongoDB ServerDB cluster. A simple cluster: single shard and three replicas, accepts defaults without changing anything.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-19-40.png)

Wait until pods are ready.<br>

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-27-40.png)

# Get credentials

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2013-31-38.png)

"Action" -> "Edit Secret" and retrieve "userAdmin" password.

# Verify connectivity

Open a terminal in one of *mongos* pods and connect using *userAdmin* credentials.

> mongo -u userAdmin<br>
```
Percona Server for MongoDB shell version v4.4.4-6
Enter password:
```
> show dbs<br>
```
mongos> show dbs
admin   0.001GB
config  0.001GB
mongos> 
```
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2014-03-31.png)

# Configure external connectivity

https://www.percona.com/doc/kubernetes-operator-for-psmongodb/expose.html

As a default, MongoDB operator creates *ClusterIP* service. Create manually *NodePort* service and remap the port on *HAProxy* node.

> oc expose deployment mydbcluster-mongos --type NodePort --name mydbcluster-mongosp<br>
> oc get svc
```
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
mydbcluster-cfg       ClusterIP   None            <none>        27017/TCP         45m
mydbcluster-mongos    ClusterIP   172.30.4.255    <none>        27017/TCP         45m
mydbcluster-mongosp   NodePort    172.30.110.20   <none>        27017:32290/TCP   22s
mydbcluster-rs0       ClusterIP   None            <none>        27017/TCP         45m
```

On *HAProxy* infrastructure node.

>  vi /etc/haproxy/haproxy.cfg 
```
rontend mgop-tcp
        bind *:27017
        default_backend mgop-tcp
        mode tcp
        option tcplog

backend mgop-tcp
        balance source
        mode tcp
        server worker0 10.17.50.214:32290 check
        server worker1 10.17.61.140:32290 check
        server worker2 10.17.62.176:32290 check

```
> systemctl reload haproxy<br>

# Verify external connection

## mongo CLI
(Assuming *boreal-inf* as HAProxy node).
> mongo mongodb://userAdmin:b3LvOkwVefeDXPXK@boreal-inf --authenticationDatabase 'admin'<br>
> show dbs
```
admin   0.001GB
config  0.001GB
mongos> 

```
# Compass GUI

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2014-35-49.png)

# Add more shards and use different Storage Class

Adding new shards require modifying *yaml* definition file.

MongoDB Operator-> All instances -> PerconsServerMongoDB -> YAML<br>

Add new entry in *spec.replsets*. You can copy and paste existing *rs0* and change name to *rs1*. In *volumeSpec* a *storageClass* is defined to bypass a default storage.<br>
```
    - affinity:
        antiAffinityTopologyKey: kubernetes.io/hostname
      arbiter:
        affinity:
          antiAffinityTopologyKey: kubernetes.io/hostname
        enabled: false
        size: 1
      expose:
        enabled: false
        exposeType: LoadBalancer
      name: rs1
      podDisruptionBudget:
        maxUnavailable: 1
      resources:
        limits:
          cpu: 300m
          memory: 0.5G
        requests:
          cpu: 300m
          memory: 0.5G
      size: 3
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: managed-nfs-storage
          resources:
            requests:
              storage: 3Gi
```
Wait until the next set of pods is created.
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2014-47-17.png)

Verify that different storage is allocated for new pods.<br>
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2014-49-02.png)

# Simple test, use sharding to allocate document in the collection to "warm" and "cold" zones.

https://docs.mongodb.com/manual/tutorial/manage-shard-zone/

Assume that shard *rs0* is "warm" and *rs1* is *cold*  (here NFS Storage Class). Old and rarely accessed document should be moved to "cold" zone maintaining cheap storage and the current document to the "warm" zone.<br>

Test environment.<br>

* two shards: *rs0* WARM and *rs1* COLD
* userdata collection: { creation_date, userid, photo_location}.  All documents older then '2020-12-31' allocated in COLD zone, and all newer in WARM zone

## Create a test database

> use testdb<br>
> db.createUser(
  {
    user: 'admin',
    pwd: 'secret',
    roles: [ { role: 'root', db: 'admin' } ]
  }
);
<br>

## Create and populate a collection

> db.userdata.insertOne({ "creation_date" : ISODate("2021-03-01"),   "userid" : 123,   "photo_location" : "example.net/storage/usr/photo_1.jpg"})<br>
> db.userdata.insertOne({ "creation_date" : ISODate("2020-10-01"),   "userid" : 123,   "photo_location" : "example.net/storage/usr/photo_99.jpg"})<br>

## Assign zone tags to shards

> sh.addShardToZone("rs0", "WARM")<br>
> sh.addShardToZone("rs1", "COLD")<br>

## Enable sharding for the collection

> sh.enableSharding("testdb")
> db.userdata.createIndex({ "creation_date"  : 1 })<br>
> sh.shardCollection( "testdb.userdata", { "creation_date" : 1 } )<br>

## Check document allocation

The shard allocation can be detected by adding *explain()* clause. Look for "shardName" field. Without shard zone enabled, the shard assignment is managed by MongoDB engine and is dependent on the current workload.

> db.userdata.find( {"creation_date" :  ISODate("2021-03-01") } ).explain()<br>
> db.userdata.find( {"creation_date" :  ISODate("2020-10-01") } ).explain()<br>

In my environment at this point, both documents are allocated to shard *rs1*.

## Enable shard zones

> sh.updateZoneKeyRange( "testdb.userdata", {"creation_date":ISODate("1970-01-01")},{"creation_date":ISODate("2020-12-31")},"COLD")<br>
> sh.updateZoneKeyRange( "testdb.userdata", {"creation_date":ISODate("2021-01-01")},{"creation_date":ISODate("2029-12-31")},"WARM")<br>

Verify.
> sh.status()
```
            { "creation_date" : ISODate("2029-12-31T00:00:00Z") } -->> { "creation_date" : { "$maxKey" : 1 } } on : rs1 Timestamp(1, 6) 
                         tag: COLD  { "creation_date" : ISODate("1970-01-01T00:00:00Z") } -->> { "creation_date" : ISODate("2020-12-31T00:00:00Z") }
                         tag: WARM  { "creation_date" : ISODate("2021-01-01T00:00:00Z") } -->> { "creation_date" : ISODate("2029-12-31T00:00:00Z") }

```

## Verify document allocation after defining shard zones

>  db.userdata.find( {"creation_date" :  ISODate("2020-10-01") } ).explain()<br>
```
				{
					"shardName" : "rs1",

```
> db.userdata.find( {"creation_date" :  ISODate("2021-03-01") } ).explain()<br>
```
				{
					"shardName" : "rs0",

```

Insert more documents to the WARM zone.

>db.userdata.insertOne({ "creation_date":ISODate("2021-03-02"), "userid" :124, "photo_location":"example.net/storage/usr/photo_2.jpg"})<br>
>db.userdata.insertOne({ "creation_date":ISODate("2021-03-03"), "userid" :125, "photo_location":"example.net/storage/usr/photo_3.jpg"})<br>
>db.userdata.insertOne({ "creation_date":ISODate("2021-03-04"), "userid" :126, "photo_location":"example.net/storage/usr/photo_4.jpg"})<br>

Verify again shard allocation using *find()*.*explain()*. *rs0* is reported.

Insert more document to COLD zone.
> db.userdata.insertOne({ "creation_date" : ISODate("2019-10-01"),   "userid" : 10,   "photo_location" : "example.net/storage/usr/photo_01.jpg"})<br>
> db.userdata.insertOne({ "creation_date" : ISODate("2018-10-01"),   "userid" : 9,   "photo_location" : "example.net/storage/usr/photo_02.jpg"})<br>
> db.userdata.insertOne({ "creation_date" : ISODate("2017-10-01"),   "userid" : 8,   "photo_location" : "example.net/storage/usr/photo_03.jpg"})<br>

Verify again shard allocation using *find()*.*explain()*. *rs1* is reported.

# Backup

https://www.percona.com/doc/kubernetes-operator-for-psmongodb/backups.html

Percona MongoDB operator provides an easy method to do backups outside OpenShift cluster. The only storage supported is Amazon S3 object store or storage compatible with Amazon S3. Backups can be done on-demand or in a scheduled fashion. <br>
Before configuring backup make sure that Amazon S3 connection data are at hand.

| Parameter | Sample value |
| ----- | ----- |
| AWS_ACCESS_KEY_ID | UkVQTEFDRS1XSVRILUFXUy1BQ0NFU1MtS0VZ
| AWS_SECRET_ACCESS_KEY | UkVQTEFDRS1XSVRILUFXUy1TRUNSRVQtS0VZ 
| S3 bucket | amhara-kops-region-store
| S3 region | us-west-2

Create a Kubernetes secret containing Amazon S3 credentials.<br>

> oc create secret generic s3-secret  --from-literal=UkVQTEFDRS1XSVRILUFXUy1BQ0NFU1MtS0VZ  --from-literal=AWS_SECRET_ACCESS_KEY=UkVQTEFDRS1XSVRILUFXUy1TRUNSRVQtS0VZ <br>

The next step is to define the backup destination in the PerconaServerMongoDB yaml file. *Installed Operator->Percona Server MongoDB Operator->PerconaServerMongoDB->YAML*
```YAML
  backup:
....

  storages:
      s3-us-west:
        s3:
          bucket: amhara-kops-region-store
          credentialsSecret: s3-secret
          region: us-west-2
        type: s3
```
Pay attention to proper indentation, otherwise, no error is reported but backup is not configured. The following elements are important:<br>
* s3-us-west: the backup storage name
* s3-secret: the OpenShift secret name created previously
* amhara-kops-region-store : Amazon S3 bucket name

The next step is to create PerconaServerMongoDBBackups instance. *Installed Operator->Percona Server MongoDB Operator->PerconaServerMongoDBBackup->Create PerconaServerMongoDBBackup*.

Switch to YAML view and create the following definition.<br>
```YAML
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBBackup
metadata:
  name: backup1
  namespace: mongodb
spec:
  psmdbCluster: mydbcluster
  storageName: s3-us-west
```
* mydbcluster: the name of the PerconaServerMongoDB instance
* s2-us-west: the backup storage name.

Create the PerconaServerMongoDBBackup instance and wait until the state is reported as ready.
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-05-06%2000-46-02.png)

If an error is reported, verify the PerconaServerMongoDBBackup yaml status section. (in the example the status reports success).
```
status:
  completed: '2021-05-05T20:14:07Z'
  destination: '2021-05-05T20:13:37Z'
  lastTransition: '2021-05-05T20:14:07Z'
  pbmName: '2021-05-05T20:13:37Z'
  s3:
    bucket: nrekha-kops-state-store
    credentialsSecret: s3-secret
    region: us-west-2
  start: '2021-05-05T20:13:54Z'
  state: ready
  storageName: s3-us-westt
```
