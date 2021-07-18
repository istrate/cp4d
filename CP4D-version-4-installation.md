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

# Change node settings

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings

As a minimum apply changes to crio and kernel setting.

Obtain *crio.conf* configuration file from any of Worker Nodes and store it in the local */tmp* directory.

> scp core@$node /etc/crio/crio.conf /tmp

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

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images
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
> oc get crd | grep operandrequest<br>
> oc api-resources --api-group operator.ibm.com
<br>

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
Install in *ibm-common-services* namespace. Mind Storage Class, here *managed-nfs-storage*

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

The last step will install CPD Control Pane. Monitor the progress:<br>

> oc get ZenService lite-cr -o jsonpath="{.status.zenStatus}{'\n'}"
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