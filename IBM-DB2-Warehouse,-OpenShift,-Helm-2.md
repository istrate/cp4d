# IBM DB2 Warehouse on OpenShift

Db2 Warehouse is designed for high-performance, in-database analytics.

https://artifacthub.io/packages/helm/ibm-charts/ibm-db2warehouse

Installing DB2 Warehouse on OpenShift is easy and described in the documentation but there are some caveats requiring more clarification.

Important: The last DB2 Warehouse release supported by Help is DB2 11.5.4. The latest DB2 11.5.5 release can be installed using OpenShift DB2 operator. https://www.ibm.com/docs/en/db2/11.5?topic=1155-installing-db2

# IBM Cloud API Access Key

Make sure you are authorized to pull IBM DB2 Warehouse docker images.

https://artifacthub.io/packages/helm/ibm-charts/ibm-db2warehouse#X-accessing-ibm-container-registry

Create new or use your existing IBM Cloud API Key. API Key is visible only once after creation, there is no way to copy it later.<br>
API Key looks like: 8Rlp7JAeTglt4N8Z69UsUv_Dc6r89F6kINK33TIpU4yu

Verify that you can authenticate in IBM registry as *iamapikey* user entering API Key as a password and pull images using credentials provided.<br>
> podman login icr.io -u iamapikey
```
Password:
Login Succeeded!
```
>  podman pull icr.io/obs/hdm/db2u/db2u.instdb:11.5.4.0-56-x86_64<br>
```
Trying to pull icr.io/obs/hdm/db2u/db2u.instdb:11.5.4.0-56-x86_64...
Getting image source signatures
Copying blob 0fa7f2d0a99c done  
Copying blob 9e28c680b472 done  
Copying blob b4b544ed1697 done  
Copying blob dc0665975713 [>-------------------------------------] 579.4KiB / 30.8MiB
Copying blob 8e3c93d02cd2 done  
Copying blob bd1ab57c0158 [>-------------------------------------] 351.4KiB / 15.7MiB
````
# Install Helm 2.17

IBM DB2 Warehouse charts are compatible with Helm 2 and Helm 3, the description here is dealing with Helm 2 only.

https://www.openshift.com/blog/getting-started-helm-openshift

Steps in a nutshell.<br>
You need *cluster-admin* authority.<br>
> oc new-project tiller<br>
> export TILLER_NAMESPACE=tiller<br>

Download and unpack Helm 2.17 packaged.<br>

> curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.17.0-linux-amd64.tar.gz | tar xz<br>

Make a link to *helm* executable in any *PATH* directories, for instance: */usr/local/bin* or *~/bin*.

> ln -s ..../helm/linux-amd64/helm <br>

Warning: at this stage, *helm version* will hang.<br>

Initialize Helm.<br>
>  helm init --stable-repo-url https://charts.helm.sh/stable --client-only<br>
> oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.17.0 | oc create -f -<br>
> oc rollout status deployment tiller<br>
```
deployment "tiller" successfully rolled out
```

Now run *helm version*.<br>
> export TILLER_NAMESPACE=tiller<br>
> helm version<br>
```
Client: &version.Version{SemVer:"v2.17.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.17.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}

```
# Install IBM DB2 Warehouse.

https://artifacthub.io/packages/helm/ibm-charts/ibm-db2warehouse

Steps in a nutshell.<br>

> git clone https://github.com/IBM/charts.git<br>
> cd charts/stable/ibm-db2warehouse/ibm_cloud_pak/pak_extensions<br>

Create a separate project.<br>
> oc new-project db2<br>

Give tiller access to *db2* project.<br>
> export TILLER_NAMESPACE=tiller<br>
> oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"<br>
```
clusterrole.rbac.authorization.k8s.io/edit added: "system:serviceaccount:tiller:tiller"
```

Create DB2 security artifacts.<br>
> ./pre-install/clusterAdministration/createSecurityClusterPrereqs.sh<br>
> ./pre-install/namespaceAdministration/createSecurityNamespacePrereqs.sh db2<br>

Create secret credentials to access IBM image repository, as *docker password* use your IBM Api Key.<br>
> oc create secret docker-registry ibm-registry --docker-server=icr.io --docker-username=iamapikey --docker-password=\<APIKey\><br>
> oc secrets link db2u ibm-registry --for=pull<br>

Pick up "DB2 release name" - it is an identifier used to create a name of different OpenShift objects, here *db2u-release*.
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-14%2015-34-07.png)

Create secrets containing DB2 and LDAP admins passwords, ${RELEASE_NAME}-db2u-ldap-bluadmin ${RELEASE_NAME}-db2u-instance

> oc create secret generic db2u-release-db2u-instance --from-literal=password=secret<br>
>oc create secret generic db2u-release-db2u-ldap-bluadmin --from-literal=password=secret<br>

The next step is to consider storage for DB2 Warehouse instance. It can be NFS storage or Rook/Ceph-filesystem.<br>

To deploy IBM DB2 Warehouse, use *db2u-install* utility. There are plenty of options available: https://artifacthub.io/packages/helm/ibm-charts/ibm-db2#X-chart-installation.

> cd common<br>
> ./db2u-install --db-type db2wh --namespace db2 --release-name db2u-release  --storage-class rook-cephfs<br>

Another approach, different Storage Class for metadata storage and Db2 storage.<br>

> cat helm_options
```
rest.enabled="false"
storage.useDynamicProvisioning="true"
storage.enableVolumeClaimTemplates="true"
storage.storageLocation.dataStorage.enablePodLevelClaim="true"
storage.storageLocation.dataStorage.enabled="true"
storage.storageLocation.dataStorage.volumeType="pvc"
storage.storageLocation.dataStorage.pvc.claim.storageClassName=rook-ceph-block
storage.storageLocation.dataStorage.pvc.claim.size="140Gi"
storage.storageLocation.metaStorage.enabled="true"
storage.storageLocation.metaStorage.volumeType="pvc"
storage.storageLocation.metaStorage.pvc.claim.storageClassName=rook-cephfs
storage.storageLocation.metaStorage.pvc.claim.size="40Gi"
```

> ./db2u-install --db-type db2wh --namespace db2  --release-name db2u-release --helm-opt-file ./helm_options<br>

# Expose Db2 Warehouse

> oc get svc<br>
```
NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S) AGE
.......
db2u-release-db2u-engn-svc   NodePort    172.30.159.211   <none>        50000:31644/TCP,50001:30406/TCP 
db2u-release-db2u-internal   ClusterIP   None             <none>        50000/TCP,9443/TCP
.......
```

In-secure NodePort: 31644 and assuming HAProxy hostname *boreal-inf*.

> vi /etc/haproxy/haproxy.cfg
```
...........
frontend db2-tcp
        bind *:50000
        default_backend db2-tcp
        mode tcp
        option tcplog

backend db2-tcp
        balance source
        mode tcp
        server worker0 10.17.50.214:31644 check
        server worker1 10.17.61.140:31644 check
        server worker2 10.17.62.176:31644 check
```

> systemctl reload haproxy<br>

# Test
Client desktop.<br>

> db2 catalog tcpip node DB2WH remote boreal-inf server 50000<br>
> db2 catalog database bludb at node db2wh<br>

Connect, admin credentials: *bluadmin/secret*<br>

> db2 connect to bludb user bluadmin<br>
```
Enter current password for bluadmin: 

   Database Connection Information

 Database server        = DB2/LINUXX8664 11.5.4.0
 SQL authorization ID   = BLUADMIN
 Local database alias   = BLUDB
```
# Install DB2 Warehouse Universal Console

Without Web UI the world of DB2 Warehouse is grey and sad.<br>

https://www.ibm.com/docs/en/db2/11.5?topic=installing-db2-unified-console

You need at least Helm 2.12 to install the console, not 2.9 as described in the documentation.<br>
The Db2 Unified Console is not a standalone application, it is paired with an already installed DB2 Warehouse instance.<br>

Collect necessary information.<br>
* db2-release-name: it is the release name of the DB2 Warehouse installed previously. Here: *db2u-release*
* console-release-name: it is the prefix added to all OpenShift Unified Console objects: Here: *db-console*

Make sure that local and remote *helm* versions match.

> export TILLER_NAMESPACE=tiller<br>
> helm version<br>
```
Client: &version.Version{SemVer:"v2.17.0", GitCommit:"a690bad98af45b015bd3da1a41f6218b1a451dbe", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.17.0", GitCommit:"a690bad98af45b015bd3da1a41f6218b1a451dbe", GitTreeState:"clean"}
```

*Helm* list should report already installed DB2 Warehouse instance.

> helm list<br>
```
NAME        	REVISION	UPDATED                 	STATUS  	CHART                    	APP VERSION	NAMESPACE
db2u-release	1       	Wed Apr 14 20:15:10 2021	DEPLOYED	ibm-db2warehouse-3.0.2   	11.5.4.0   	db2   
```

> cd charts/stable/ibm-unified-console/ibm_cloud_pak/pak_extensions<br>
> ./deploy_console.sh --db-release-name  db2u-release --console-release-name db-console<br>

Wait several minutes until all *Univeral Console* pods are up and ready.<br>

>  oc get pods
```
NAME                                                  READY   STATUS    RESTARTS   AGE
db-console-ibm-unified-console-api-6c97db5fcc-8m99k   1/1     Running   0          24m
db-console-ibm-unified-console-ui-6c54797f66-464xm    1/1     Running   0          24m
```

Identify Unified Console *route* address, here *db-console-db2.apps.boreal.cp.fyre.ibm.com* <br>

> oc get route<br>
```
NAME         HOST/PORT                                    PATH   SERVICES                            PORT    TERMINATION   WILDCARD
db-console   db-console-db2.apps.boreal.cp.fyre.ibm.com          db-console-ibm-unified-console-ui   https   passthrough   None
```

Open https://db-console-db2.apps.boreal.cp.fyre.ibm.com  (secure port)

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-04-21%2001-04-05.png)
<br>
User/password: bluadmin/{password}
