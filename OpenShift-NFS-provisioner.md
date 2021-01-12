# OpenShift NFS provisioner

This page is an adaptation of:<br>
https://medium.com/faun/openshift-dynamic-nfs-persistent-volume-using-nfs-client-provisioner-fcbb8c9344e

# Prerequisities

Install and configure the NFS server. Make sure that NFS host is visible and NFS volume can be mounted from all nodes in OpenShift cluster. Prepare NFS mount parameters. Example:

* NFS server: 10.16.65.240
* NFS mount point: /data/nfs2

# OpenShift project

You can use a default project or create a separate project to keep NFS related objects. In this example, *nfs-storage* project is used.<br>

> oc new-project nfs-storage<br>

# Create ServiceAccount and Role

 > curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/serviceaccount.yaml | sed "s@: default@: nfs-storage@g" | oc create -f -
```
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```

If different then *nfs-storage*, replace *nfs-storage* in the command below with project used (for instance: default).<br>

>  oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:nfs-storage:nfs-client-provisioner
```
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hostmount-anyuid added: "nfs-client-provisioner"
```

# Deploy NFS provisioner application

Replace NFS mount data according to your environment.<br>

> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/deployment.yaml |  sed -e "s@10.10.10.60@10.16.65.240@g" | sed -e "s@/ifs/kubernetes@/data/nfs2@g" | oc create -f -
```
deployment.apps/nfs-client-provisioner created
```

> oc get pods<br>
```
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-59b865db57-6bf89   1/1     Running   0          38s
```
# Create StorageClass

> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/class.yaml
```
storageclass.storage.k8s.io/managed-nfs-storage created
```

> oc get sc<br>

```
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  35s
```

To make *managed-nfs-storage* a default class, add *is-default-class* annotation.

```
annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
```

> oc edit sc managed-nfs-storage<br>
```
....
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  creationTimestamp: "2020-12-31T23:50:45Z"
  managedFields:
```

> oc get sc<br>
```
NAME                            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   fuseim.pri/ifs   Delete          Immediate           false                  19m
```

# Test
## Create test PVC

> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-claim.yaml
```
persistentvolumeclaim/test-claim created
```

Make sure that *pvc* is bounded, *Bound* value in *STATUS* column.<br>

> oc get pvc<br>

```
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-2fd9aeaf-4bff-49a9-b6c4-187fe91ce820   1Mi        RWX            managed-nfs-storage   43s
```
## Create a test application

> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-pod.yaml
```
pod/test-pod created
```

The *Completed* value in *STATUS* column is expected.<br>

> oc get pods<br>
```
NAME                                      READY   STATUS      RESTARTS   AGE
test-pod                                  0/1     Completed   0          4m19s
```

Logon to NFS server host and verify the SUCCESS file.<br>
> ll /data/nfs2/nfs-storage-test-claim-pvc-2fd9aeaf-4bff-49a9-b6c4-187fe91ce820/
```
-rw-r--r-- 1 root root 0 12-31 15:55 SUCCESS
```

> oc delete pod/test-pod<br>
> oc delete pvc/test-claim<br>

# Create another StorageClass using different NFS server

## Create

In order to utilize another NFS server, create a similar Deployment and StorageClass but using different names.<br>

Assume:<br>
* Deployment name: nfs-zen-provisioner
* StorageClass name: managed-zen-storage
* NFS host: 9.30.97.206
* Mount point: /data/nfs

Use the same *nfs-storage* project.<br>

> oc project nfs-storage<br>

Download Deployment yaml.<br>
> wget https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/deployment.yaml<br>

Modify manually.<br>
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-zen-provisioner
  labels:
    app: nfs-zen-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-zen-provisioner
  template:
    metadata:
      labels:
        app: nfs-zen-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-zen-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/zen
            - name: NFS_SERVER
              value: 9.30.97.206
            - name: NFS_PATH
              value: /data/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: 9.30.97.206
            path: /data/nfs

```

Create *nfs-zen-provisioner* deployment.
> oc create -f deployment.yaml<br>
```
deployment.apps/nfs-zen-provisioner created
```
> oc get pods<br>
```
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-59b865db57-6bf89   1/1     Running   0          13h
nfs-zen-provisioner-b796d9774-dxf4x       1/1     Running   0          8s
```

Create StorageClass.<br>
> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/class.yaml | sed s@nfs-@zen-@g | sed s@ifs@zen@g | oc create -f -

```
storageclass.storage.k8s.io/managed-zen-storage created
```
> oc get sc<br>
```
NAME                            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   fuseim.pri/ifs   Delete          Immediate           false                  13h
managed-zen-storage             fuseim.pri/zen   Delete          Immediate           false                  14s
```

## Test

The same test as above but modify the claimed StorageClass.<br>

> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-claim.yaml | sed s@-nfs@-zen@g | oc create -f -<br>
> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-pod.yaml

Logon to *9.30.97.206* NFS server.<br>
> ll /data/nfs/nfs-storage-test-claim-pvc-854785ef-1267-4bb1-b64e-9378521e4b41/<br>
```
-rw-r--r-- 1 root root 0 01-01 05:42 SUCCESS
```
> oc delete pod/test-pod<br>
> oc delete pvc/test-claim<br>