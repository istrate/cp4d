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
# Projects

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-projects-namespaces

> oc new-project ibm-common-services<br>
> oc new-project cpd-instance<br>

# Operator Group

Create Operator Group (Express installation).

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

# Create CatalogSource

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images<br>
(3. Creating the catalog source)

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
# Create Operator Subscriptions

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ibm-common-services
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

Verify the status of the operators:
> oc --namespace ibm-common-services get csv<br>
```
ibm-common-service-operator.v3.9.0            IBM Cloud Pak foundational services    3.9.0     ibm-common-service-operator.v3.8.0            Succeeded
ibm-namespace-scope-operator.v1.3.0           IBM NamespaceScope Operator            1.3.0     ibm-namespace-scope-operator.v1.2.0           Succeeded
operand-deployment-lifecycle-manager.v1.7.0   Operand Deployment Lifecycle Manager   1.7.0     operand-deployment-lifecycle-manager.v1.6.0   Succeeded
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
ibm-common-service-operator-d656945f9-zrdft             1/1     Running   0          3m56s
ibm-common-service-webhook-79bb458694-h9j4m             1/1     Running   0          3m21s
ibm-namespace-scope-operator-5d869759c9-6trpl           1/1     Running   0          3m29s
operand-deployment-lifecycle-manager-5c74799b47-xxhc8   1/1     Running   0          2m57s
secretshare-7c9fdc588d-d8n8b                            1/1     Running   0          3m15s
```
#  Creating an operator subscription for the scheduling service

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  annotations:
  labels:
    operators.coreos.com/ibm-cpd-scheduling-operator.aks: ""
    velero.io/exclude-from-backup: "true"
  name: ibm-cpd-scheduling-catalog-subscription
  namespace: ibm-common-services    # Pick the project that contains the Cloud Pak for Data operator
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: ibm-cpd-scheduling-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```
# Installing the scheduling service

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=service-installing-scheduling
<br>

Assign RBAC role.<br>

> oc adm policy add-cluster-role-to-user system:kube-scheduler system:serviceaccount:ibm-common-services:ibm-cpd-scheduling-operator 
--rolebinding-name=ibm-cpd-scheduling-operator-kube-sched-crb-ibm-common-services<br>


Install in *ibm-common-services* namespace. Consider *storageClass*, here *managed-nfs-storage*

```
cat <<EOF |oc apply -f -
apiVersion: scheduler.spectrumcomputing.ibm.com/v1
kind: Scheduling
metadata:
  labels:
    release: cpd-scheduler
    velero.io/exclude-from-backup: "true"
  name: ibm-cpd-scheduler
  namespace: ibm-common-services   
spec:
  appVersion: 1.2.1
  version: 1.2.1
  cluster:
    pvc:
      dynamicStorage: true
      size: 10G
  license:
    accept: true
    license: Standard       # Specify the license you purchased
  registry: cp.icr.io/cp/cpd
  releasename: ibm-cpd-scheduler
  storageClass: managed-nfs-storage     # See the guidance in "Information you need to complete this task"
  scheduler:
    image: ibm-cpd-scheduler
    imagePullPolicy: Always
    replicas: 1
    resources:
      limits:
        cpu: "1"
        memory: 4G
      requests:
        cpu: "1"
        memory: 4G
  agent:
    image: ibm-cpd-scheduler-agent
    imagePullPolicy: Always
    resources:
      limits:
        cpu: 200m
        memory: 750M
      requests:
        cpu: 200m
        memory: 750M
  mwebhook:
    image: ibm-cpd-scheduler-mutate-webhook
    imagePullPolicy: Always
    replicas: 1
    resources:
      limits:
        cpu: 200m
        memory: 1G
      requests:
        cpu: 200m
        memory: 1G
  vwebhook:
    image: ibm-cpd-scheduler-webhook
    imagePullPolicy: Always
    replicas: 1
    resources:
      limits:
        cpu: 200m
        memory: 1G
      requests:
        cpu: 200m
        memory: 1G
EOF
```
<br>

> oc get scheduling ibm-cpd-scheduler -n ibm-common-services -o yaml<br>

If successful, there will be status reported like below:<br>
```
status:
  conditions:
  - message: Running reconciliation
    reason: Running
    status: "True"
    type: Running
  cpd-schedulingStatus: Completed
  type: Ready
  versions:
    reconciled: 1.2.1

```

> oc get pod -n ibm-common-services | grep scheduler
```
ibm-cpd-scheduler-agent-r24qz                           1/1     Running   0          5m28s
ibm-cpd-scheduler-agent-rwxh9                           1/1     Running   0          5m28s
ibm-cpd-scheduler-agent-sfxqc                           1/1     Running   0          5m28s
ibm-cpd-scheduler-agent-whrvv                           1/1     Running   0          5m28s
ibm-cpd-scheduler-agent-xsnc2                           1/1     Running   0          5m28s
ibm-cpd-scheduler-agent-zhnd5                           1/1     Running   0          5m28s
ibm-cpd-scheduler-mutating-webhook-68b4d56c5c-p2dvk     1/1     Running   0          4m40s
ibm-cpd-scheduler-scheduler-57f5d56758-8jhkr            1/1     Running   0          5m25s
ibm-cpd-scheduler-webhook-f4cc67567-pxg4h               1/1     Running   0          5m23s
```
# Install CPD platform operator

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ibm-common-services
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

Install Catalog Source: CSS Common Core Services
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-ccs-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-ccs-operator-catalog@sha256:34854b0b5684d670cf1624d01e659e9900f4206987242b453ee917b32b79f5b7
  imagePullPolicy: Always
  displayName: CPD Common Core Services
  publisher: IBM
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```
Install Catalog Source: Data Refinery
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-datarefinery-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-datarefinery-operator-catalog@sha256:27c6b458244a7c8d12da72a18811d797a1bef19dadf84b38cedf6461fe53643a
  imagePullPolicy: Always
  displayName: Cloud Pak for Data IBM DataRefinery
  publisher: IBM
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

Install more dependent Catalog Sources.
```
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-db2aaservice-cp4d-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-db2aaservice-cp4d-operator-catalog@sha256:a0d9b6c314193795ec1918e4227ede916743381285b719b3d8cfb05c35fec071
  imagePullPolicy: Always
  displayName: IBM Db2aaservice CP4D Catalog
  publisher: IBM
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-iis-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-iis-operator-catalog@sha256:3ad952987b2f4d921459b0d3bad8e30a7ddae9e0c5beb407b98cf3c09713efcc
  imagePullPolicy: Always
  displayName: CPD IBM Information Server
  publisher: IBM
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-wml-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: Cloud Pak for Data Watson Machine Learning
  publisher: IBM
  sourceType: grpc
  imagePullPolicy: Always
  image: icr.io/cpopen/ibm-cpd-wml-operator-catalog@sha256:d2da8a2573c0241b5c53af4d875dbfbf988484768caec2e4e6231417828cb192
  updateStrategy:
    registryPoll:
      interval: 45m
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-ws-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-ws-operator-catalog@sha256:bf6b42df3d8cee32740d3273154986b28dedbf03349116fba39974dc29610521
  imagePullPolicy: Always
  displayName: CPD IBM Watson Studio
  publisher: IBM
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencontent-elasticsearch-dev-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/opencontent-elasticsearch-operator-catalog@sha256:bc284b8c2754af2eba81bb1edf6daa59dc823bf7a81fe91710c603f563a9a724
  displayName: IBM Opencontent Elasticsearch Catalog
  publisher: CloudpakOpen
  updateStrategy:
    registryPoll:
      interval: 45m
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-rabbitmq-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: IBM RabbitMQ operator Catalog
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/opencontent-rabbitmq-operator-catalog@sha256:c3b14816eabc04bcdd5c653eaf6e0824adb020ca45d81d57059f50c80f22964f
  updateStrategy:
    registryPoll:
      interval: 45m
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cloud-databases-redis-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: ibm-cloud-databases-redis-operator-catalog
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-cloud-databases-redis-catalog@sha256:980e4182ec20a01a93f3c18310e0aa5346dc299c551bd8aca070ddf2a5bf9ca5
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-cpd-ws-runtimes-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: icr.io/cpopen/ibm-cpd-ws-runtimes-operator-catalog@sha256:c1faf293456261f418e01795eecd4fe8b48cc1e8b37631fb6433fad261b74ea4
  imagePullPolicy: Always
  displayName: CPD Watson Studio Runtimes
  publisher: IBM
EOF
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

The last step will install CPD Control Pane. It will take several minutes to create *ZenService* kind and *lite-cr* instance. Monitor the progress:<br>

> oc get ZenService lite-cr -o jsonpath="{.status.zenStatus}{'\n'}"
```
error: the server doesn't have a resource type "ZenService"
```
```
Error from server (NotFound): zenservices.zen.cpd.ibm.com "lite-cr" not found
```
```
InProgress
```

It will take 10-20 minutes until completed.
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

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-creating-operator-subscriptions

Create subscription<br>
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  annotations:
  name: ibm-cpd-ws-operator-catalog-subscription
  namespace: ibm-common-services
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: ibm-cpd-wsl
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF
```

Wait until resource is ready.<br>
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
  version: 4.0.0
  storageClass: managed-nfs-storage
  docker_registry_prefix: cp.icr.io/cp/cpd
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

# Watson Knowledge Studio and DB2 prerequisites

## Adjust kernel settings
https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/cpd/install/node-settings.html#concept_vcl_pfg_tpb__kernel#concept_vcl_pfg_tpb__kernel

(Assuming 64GB Worker Nodes)

```
cat << EOF | oc apply -f -
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
## Additional kernel settings for DB2

> oc label machineconfigpool worker db2u-kubelet=sysctl

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

# Watson Knowledge Catalog

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=catalog-installing-watson-knowledge



## Create Custom CSS

> vi wkc-iis-scc.yaml
```
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: WKC/IIS provides all features of the restricted SCC
      but runs as user 10032.
  name: wkc-iis-scc
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: MustRunAs
  uid: 10032
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
users:
- system:serviceaccount:cpd-instance:wkc-iis-sa
```
> oc create -f wkc-iis-scc.yaml<br>

Verify<br>

> oc get scc wkc-iis-scc<br>
```
NAME          PRIV    CAPS         SELINUX     RUNASUSER   FSGROUP    SUPGROUP   PRIORITY     READONLYROOTFS   VOLUMES
wkc-iis-scc   false   <no value>   MustRunAs   MustRunAs   RunAsAny   RunAsAny   <no value>   false            ["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]
```
## Create WKC Operator Subscription
```
cat << EOF | oc apply -f -
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
## Install Python2 and yaml dependency

> yum install -y python2<br>
> alternatives --set python /usr/bin/python2<br>
> pip2 install pyyaml<br>

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
  version: 4.0.0
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