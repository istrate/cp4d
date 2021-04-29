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

Open terminal in one of *mongos* pods and connect using *userAdmin* credentials.

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
> cps$  mongo mongodb://userAdmin:b3LvOkwVefeDXPXK@boreal-inf --authenticationDatabase 'admin'<br>
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
|[](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-29%2014-49-02.png)


