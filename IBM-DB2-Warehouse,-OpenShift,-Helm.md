# IBM DB2 Warehouse on OpenShift

Db2 Warehouse is designed for high-performance, in-database analytics.

https://artifacthub.io/packages/helm/ibm-charts/ibm-db2warehouse

Installing DB2 Warehouse on OpenShift is easy and described in the documentation but there are some caveats requiring more clarification.

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
# Install Helm 2.9

IBM DB2 Warehouse charts are compatible with Helm 2.x only. 

https://www.openshift.com/blog/getting-started-helm-openshift

Steps in a nutshell.<br>
You need *cluster-admin* authority.<br>
> oc new-project tiller<br>
> export TILLER_NAMESPACE=tiller<br>

Download and unpack Helm 2.9 packaged.<br>

> curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.0-linux-amd64.tar.gz | tar xz<br>

Make a link to *helm* executable in any *PATH* directories, for instance: */usr/local/bin* or *~/bin*.

> ln -s ..../helm/linux-amd64/helm <br>

Warning: at this stage, *helm version* will hang.<br>

Initialize Helm.<br>
>  helm init --stable-repo-url https://charts.helm.sh/stable --client-only<br>
> oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.9.0 | oc create -f -<br>
> oc rollout status deployment tiller<br>
```
deployment "tiller" successfully rolled out
```

Now run *helm version*.<br>
> export TILLER_NAMESPACE=tiller<br>
> helm version<br>
```
Client: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}

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

