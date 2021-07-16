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

Apply at least two change to */tmp/crio.conf* file<br>

> vi /tmp/crio.conf
```
[crio.runtime]

default_ulimits = [
  "nofile=66536:66536"
]





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
* Username: oc
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
