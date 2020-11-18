# OpenShift routes

https://docs.openshift.com/container-platform/4.5/networking/routes/route-configuration.html

OpenShift is an extension to Kubernetes service allowing external access to OpenShift/Kubernetes applications. But it requires at least one of the OpenShift nodes to be accessible directly from the client node.<br>

# Architecture

Assume the architecture where the whole OpenShift cluster is running on a private network and the gateway is a separate node running in both networks, private and public, but not being included in the OpenShift cluster

| Node | Network | Role 
| --- | ---- | ---- 
| master0.oc.com | private | Master
| worker0.oc.com | private | Worker
| worker1.oc.com | private | Worker
| worker2.oc.com | private | Worker
| inf.oc.com | private, public | Gateway to the cluster, HAProxy, NFS services
| client.sb.com | public | Client desktop

The client can access the gateway node and run OpenShift console or *oc* command utilizing the HAProxy gateway node service.

# Create MySQL instance

<br>
> oc new-app --as-deployment-config  --docker-image=registry.access.redhat.com/rhscl/mysql-57-rhel7:latest --name=mysql-openshift  -e MYSQL_USER=user1 -e MYSQL_PASSWORD=mypa55 -e MYSQL_DATABASE=testdb  -e MYSQL_ROOT_PASSWORD=r00tpa55<br>

This command creates MySQL instance and MySQL service.
<br>
> oc get svc
```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql-openshift   ClusterIP   172.30.94.36   <none>        3306/TCP   4d12h
sbartkowski:cpd$ 
```
<br>
Make sure that MySQL database is up and running.
> oc get pods
```
NAME                      READY   STATUS    RESTARTS   AGE
mysql-openshift-1-85ft6   1/1     Running   1          26h
```
> oc port-forward mysql-openshift-1-85ft6  3306:3306<br>



# Create a route.

> oc expose service mysql-openshift<br>
> of get route<br>
```
NAME              HOST/PORT                                       PATH   SERVICES          PORT       TERMINATION   WILDCARD
mysql-openshift   mysql-openshift-sb.apps.rumen.os.fyre.ibm.com          mysql-openshift   3306-tcp                 None

```




