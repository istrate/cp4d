# Introduction

This article describes the minimal steps to install Cloud Pak for Data version 4 installation on OpenShift cluster. It is using IBM Public Repositories to download CP4D images.

Source of truth: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=installing

Storage: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-setting-up-shared-persistent-storage

NFS storage layer: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=storage-setting-up-nfs

Also: https://github.com/stanislawbartkowski/CP4D/wiki/OpenShift-NFS-provisioner

# Prerequsities

* OpenShift 4.6 at least. Minimalistic configuration: 3 Master Node and 3 Worker Node (16 CPU, 32 GB RAM)
* Storage. Any supported storage (link above). The simplest storage configuration to start with is NFS Provisioner
* CP4D entitlement, API Key: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-obtaining-your-entitlement-api-key
* Authorized OpenShift *admin-cluster* user

# Test your API Key

Validate the API Key before running the installation. As password, copy and paste your API Key.
> podman login cp.icr.io<br>
```
Username: cp
Password: 
Login Succeeded!
```

Download one of the images as an access test.

> podman pull icr.io/cpopen/ibm-cpd-ccs-operator-catalog@sha256:34854b0b5684d670cf1624d01e659e9900f4206987242b453ee917b32b79f5b7
```
Trying to pull icr.io/cpopen/ibm-cpd-ccs-operator-catalog@sha256:34854b0b5684d670cf1624d01e659e9900f4206987242b453ee917b32b79f5b7...
Getting image source signatures
Checking if image destination supports signatures
Copying blob f557be5d7c54 done  
Copying blob be14f6f66b43 done  
Copying blob ee27fede83dd done  
Copying blob 48182b3ae68a done  
Copying blob ac7bbcb20bdc done  
Copying blob 426f520a254b done  
Copying blob 46d075a4a2fd done  
Copying config 1a2ab84f3c done  
Writing manifest to image destination
Storing signatures
1a2ab84f3ce3230e867c250641633e37259b5f651f1d4cbb7d84c27221377806
```

# Change node settings

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings

As a minimum apply changes to crio and kernel setting.

Obtain *crio.conf* configuration file from any of Worker Nodes and store it in the local */tmp* directory.

> scp core@\<node\>:/etc/crio/crio.conf /tmp

Make at least two changes to */tmp/crio.conf* file<br>

> vi /tmp/crio.conf
```
[crio.runtime]

default_ulimits = [
  "nofile=66536:66536"
]
..............
..............
# Maximum number of processes allowed in a container.
pids_limit = 12288
```

Apply reconfigured *crio* to the cluster.<br>
```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 51-worker-cp4d-crio-conf
spec:
  config:
    ignition:
      version: 3.1.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,$(cat /tmp/crio.conf | base64 -w0)
        filesystem: root
        mode: 0644
        path: /etc/crio/crio.conf
EOF
```

Wait until all nodes are ready again.<br>

> oc get nodes
```
NAME                             STATUS                     ROLES    AGE   VERSION
master0.starry.cp.fyre.ibm.com   Ready                      master   14h   v1.20.0+87cc9a4
master1.starry.cp.fyre.ibm.com   Ready                      master   14h   v1.20.0+87cc9a4
master2.starry.cp.fyre.ibm.com   Ready                      master   14h   v1.20.0+87cc9a4
worker0.starry.cp.fyre.ibm.com   Ready                      worker   14h   v1.20.0+87cc9a4
worker1.starry.cp.fyre.ibm.com   Ready                      worker   14h   v1.20.0+87cc9a4
worker2.starry.cp.fyre.ibm.com   Ready,SchedulingDisabled   worker   14h   v1.20.0+87cc9a4
```

When it is completed, the command should report all nodes UPDATED and not UPDATING.

> oc get mcp
```
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-434719fcbfb7ad75acf961087a4fc7d0   True      False      False      3              3                   3                     0                      15h
worker   rendered-worker-fb5f23408c52da40a27de27455657022   True      False      False      3              3                   3                     0                      15h
```

# Configure cluster pull secrets to download CP4D artefacts

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images

Obtain CP4D entitlement, the API Key looks like: *eyghbGciOiPOUzI1NiJ9.eyJcvbMiOiJJQk0gVCFya2V0cGxhY2UiLCJpYXQiOjE1OAcENTM2PDPsImp0aSI6IjM0YWU1ZjlkYjc3MjQ3NTM4OWIyOTFhMjY3YWQ3OTE0In0.ww83p2eBNIpVon13UQXTqJP4nBy_piNh6cPi-3oR0ylZ*

You can configure cluster pull secrets using the *oc* command line or through the OpenShift console.

OpenShift console -> Projects -> openshift-config -> Secrets -> pull-secret -> Actions -> Edit Secret -> Add Credentials<br>

Add new secret using data:
* Registry Server Address: cp.icr.io
* Username: cp
* Password: your API KEY as plain-text, not base64 encrypted

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-07-16%2012-13-22.png)

# Projects

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-projects-namespaces

> oc new-project ibm-common-services<br>
> oc new-project cpd-instance<br>

# Create the CatalogSources

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images<br>

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: "IBM Operator Catalog" 
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-operator-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

Optional: DB2 CatalogSource, a prerequisite for other CP4D components
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-db2uoperator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: docker.io/ibmcom/ibm-db2uoperator-catalog:latest
  imagePullPolicy: Always
  displayName: IBM Db2U Catalog
  publisher: IBM
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

# Operator Group

Create Operator Group

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ibm-common-services
spec:
  targetNamespaces:
  - ibm-common-services
EOF
```

# Create Operator Subscriptions

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ibm-common-services
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance:  ibm-cpd-wkc-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-wkc-operator
    app.kubernetes.io/name:  ibm-cpd-wkc-operator-catalog-subscription
  name: ibm-cpd-wkc-operator-catalog-subscription
  namespace: ibm-common-services
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-wkc
    source: ibm-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
```
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-datastage-operator
  namespace: ibm-common-services
spec:
  channel: v1.0
  installPlanApproval: Automatic
  name: ibm-datastage-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-dv-operator-catalog-subscription
  namespace: ibm-common-services
spec:
  channel: v1.7
  installPlanApproval: Automatic
  name: ibm-dv-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

Verify the status of the operators:
> oc --namespace ibm-common-services get csv<br>
```
NAME                                  DISPLAY                               VERSION   REPLACES                              PHASE
cpd-platform-operator.v2.0.3          Cloud Pak for Data Operator           2.0.3                                           Succeeded
ibm-common-service-operator.v3.11.0   IBM Cloud Pak foundational services   3.11.0    ibm-common-service-operator.v3.10.0   Succeeded
ibm-cpd-wkc.v1.0.1                    WKC Services                          1.0.1     ibm-cpd-wkc.v1.0.0                    Succeeded
ibm-datastage-operator.v1.0.1         IBM DataStage                         1.0.1     ibm-datastage-operator.v1.0.0         Succeeded
ibm-dv-operator.v1.7.1                IBM Data Virtualization               1.7.1     ibm-dv-operator.v1.7.0                Succeeded
```

> oc get crd | grep operandrequest<br>
```
operandrequests.operator.ibm.com                                  2021-07-26T11:57:51Z
```

> oc api-resources --api-group operator.ibm.com
```
commonservices                   operator.ibm.com/v3         true         CommonService
namespacescopes     nss          operator.ibm.com/v1         true         NamespaceScope
operandbindinfos    opbi         operator.ibm.com/v1alpha1   true         OperandBindInfo
operandconfigs      opcon        operator.ibm.com/v1alpha1   true         OperandConfig
operandregistries   opreg        operator.ibm.com/v1alpha1   true         OperandRegistry
operandrequests     opreq        operator.ibm.com/v1alpha1   true         OperandRequest
podpresets                       operator.ibm.com/v1alpha1   true         PodPreset
```
<br>

> oc get pod -n ibm-common-services
```
cpd-platform-operator-manager-64d6d77bf7-zfdcq          1/1     Running   0          2m31s
ibm-common-service-operator-6c4fcdb885-wscqb            1/1     Running   0          2m30s
ibm-common-service-webhook-bd65548d7-wzpmg              1/1     Running   0          64s
ibm-cpd-wkc-operator-5c454c686c-gcz45                   1/1     Running   0          2m30s
ibm-datastage-operator-59fd75f575-x6dpk                 1/1     Running   0          2m33s
ibm-dv-operator-controller-manager-fc4946d5-wwjlj       1/1     Running   0          2m9s
ibm-namespace-scope-operator-747fd7dc-n7g8p             1/1     Running   0          71s
operand-deployment-lifecycle-manager-59d99c8f9d-2xfk2   1/1     Running   0          38s
secretshare-5fd5b4fb76-fn225                            1/1     Running   0          58s
```

# Install Cloud Pak for Data 

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=installing-cloud-pak-data

Install the operand request.<br>
```
cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: empty-request
  namespace: cpd-instance
spec:
  requests: []
EOF
```

Install Cloud Pak For Data control plane. Pay attention to:
* License (here *Standard*)
* Storage Class (here *managed-nfs-storage*)

```
cat <<EOF |oc apply -f -
apiVersion: cpd.ibm.com/v1
kind: Ibmcpd
metadata:
  name: ibmcpd-cr
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Standard
  storageClass: managed-nfs-storage
  zenCoreMetadbStorageClass: managed-nfs-storage
  version: "4.0.1"
EOF
```

The last step will install CPD Control Pane.<br>

> oc get Ibmcpd ibmcpd-cr -o jsonpath='{.status.controlPlaneStatus}'
```
error: the server doesn't have a resource type "Ibmcpd"
```
```
Error from server (NotFound): ibmcpd.zen.cpd.ibm.com "ibmcpd-cr" not found
```
```
InProgress
```

It will take up to 1 hour until completed.
```
Completed
```

# Enter Cloud Pak for Data Console

Get URL

> oc get ZenService lite-cr -o jsonpath="{.status.url}{'\n'}"<br>
```
cpd-cpd-instance.apps.starry.cp.fyre.ibm.com
```

Get the initial password

> oc extract secret/admin-user-details --keys=initial_admin_password --to=-
```
# initial_admin_password
gNKvIwRDpbLk
```
# Install Watson Studio

Make sure WS resource is ready.<br>

> oc get WS<br>
```
error: the server doesn't have a resource type "WS"
```
```
No resources were found in cpd-instance namespace.
```

Create a custom resource. Apply proper license (here Standard) and Storage Class (here managed-nfs-storage) <br>
```
cat <<EOF |oc apply -f -
apiVersion: ws.cpd.ibm.com/v1beta1
kind: WS
metadata:
  name: ws-cr
  namespace: cpd-instance
spec:
  license:
    accept: true
    license: Standard
  version: 4.0.1
  storageClass: managed-nfs-storage
EOF
```
Monitor the installation progress. It can take 1-2 hours until finished.<br>
> oc get WS ws-cr -o jsonpath='{.status.wsStatus} {"\n"}'<br>
```
InProgress
```
```
Completed 

```
Open Cloud Pak for Data console and navigate to "Services Catalog". Watson Studio and Data Refinery tiles should be marked as *Enabled*.
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-07-27%2014-28-18.png)

# Watson Knowledge Catalog and DB2 prerequisites

Tuned resource, prereq for DB2. The parameters are aligned assuming 64GB RAM. In case of a more tiny environment, adjust accordingly.
```
cat <<EOF |oc apply -f -
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: cp4d-wkc-ipc
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - name: cp4d-wkc-ipc
    data: |
      [main]
      summary=Tune IPC Kernel parameters on OpenShift Worker Nodes running WKC Pods
      [sysctl]
      kernel.shmall = 33554432
      kernel.shmmax = 68719476736
      kernel.shmmni = 32768
      kernel.sem = 250 1024000 100 32768
      kernel.msgmax = 65536
      kernel.msgmnb = 65536
      kernel.msgmni = 32768
      vm.max_map_count = 262144
  recommend:
  - match:
    - label: node-role.kubernetes.io/worker
    priority: 10
    profile: cp4d-wkc-ipc
EOF
```

Whitelist unsafe sysctl, DB2 prereq
```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: db2u-kubelet
spec:
  machineConfigPoolSelector:
    matchLabels:
      db2u-kubelet: sysctl
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF
```
Label all worker nodes as unsafe sysctl whilelisted.
```
oc label machineconfigpool worker db2u-kubelet=sysctl
```


# Watson Knowledge Catalog

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=catalog-installing-watson-knowledge

## Make sure that WKC kind is created

>  oc get WKC<br>

```
No resources found in cpd-instance namespace.
```

## Install Watson Knowledge Catalog

Consider *storageClass* (here managed-nfs-storage) and license (here Standard)

```
cat << EOF | oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
metadata:
  name: wkc-cr
  namespace: cpd-instance
spec:
  version: 4.0.1
  license:
    accept: true
    license: Standard
  storageClass: managed-nfs-storage
  useODLM: true
  docker_registry_prefix: cp.icr.io/cp/cpd 
EOF
```

Monitor the progress<br>

> oc get WKC wkc-cr -o jsonpath='{.status.wkcStatus} {"\n"}'<br>
```
InProgress 
```
There is also a lot of dependencies installed together with Watson Knowledge Catalog.

> oc get CCS ccs-cr -o jsonpath='{.status.ccsStatus} {"\n"}'<br>
> oc get DataRefinery datarefinery-sample -o jsonpath='{.status.datarefineryStatus} {"\n"}'<br>
> oc get Db2aaserviceService db2aaservice-cr -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'<br>
> oc get IIS iis-cr -o jsonpath='{.status.iisStatus} {"\n"}'<br>
> oc get UG ug-cr -o jsonpath='{.status.ugStatus} {"\n"}'<br>

# HEE - Hadoop Execution Engine

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=ieeah-installing-execution-engine-apache-hadoop

Install operator subscription.
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance: ibm-cpd-hadoop-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-hadoop-operator
    app.kubernetes.io/name: ibm-cpd-hadoop-operator-catalog-subscription
  name: ibm-cpd-hadoop-operator-catalog-subscription
  namespace: ibm-common-services
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-hadoop
    source: ibm-operator-catalog
    sourceNamespace: openshift-marketplace
EOF
```

Wait until *Hadoop* resource is available.

> oc get Hadoop<br>
```
error: the server doesn't have a resource type "Hadoop"
```
```
No resources found in cpd-instance namespace.
```

Install Customer Resource. Consider license (here *Standard*) and Storage Class (here *nfs-managed-storage*)
```
cat <<EOF |oc apply -f -
apiVersion: hadoop.cpd.ibm.com/v1
kind: Hadoop
metadata:
  name: hadoop-cr  
  namespace: cpd-instance 
spec:
  docker_registry_prefix: cp.icr.io/cp/cpd
  size: small
  license:
    accept: true
    license: Standard
  version: 4.0.0
  storageVendor: ocs
  storageClass: managed-nfs-storage
EOF

```

Monitor the progress, it takes several minutes before completing.

> oc get Hadoop hadoop-cr -o jsonpath='{.status.hadoopStatus} {"\n"}'
```
Completed
```
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-02%2014-21-11.png)


