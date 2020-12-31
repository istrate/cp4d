# OpenShift NFS provisioner

This page is an adaptation of https://medium.com/faun/openshift-dynamic-nfs-persistent-volume-using-nfs-client-provisioner-fcbb8c9344e

# Prerequisities

Install and configure the NFS server. Make sure that NFS host is visible and NFS volume can be mounted from all nodes in OpenShift cluster. Prepare NFS mount parameters. Example:

* NFS server: 10.16.65.240
* NFS mount point: /data/nfs2

# OpenShift project

You can use a default project or create a separate project to keep NFS related objects. Here *nfs-storage* project is used.<br>

> oc new-project nfs-storage<br>

# Create serviceaccount and role

 > curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/serviceaccount.yaml | sed "s@: default@: nfs-storage@g" | oc create -f -
```
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```

Replace *nfs-storage* in the command below with project used (for instance: default).<br>

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
# Create Storage Class

> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/class.yaml | oc create -f -
```
storageclass.storage.k8s.io/managed-nfs-storage created
```

> oc get sc
```
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  35s
```
```
# Test
## Create test PVC

> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-claim.yaml | oc create -f -<br>
```
persistentvolumeclaim/test-claim created
```

Make sure that *pvc* is bounded, *Bound* value in *STATUS* column.<br>

> oc get pvc<br>
```NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-2fd9aeaf-4bff-49a9-b6c4-187fe91ce820   1Mi        RWX            managed-nfs-storage   43s
```
## Create test application

> curl -s https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/nfs-storage/test-pod.yaml | oc create -f -<br>
```
pod/test-pod created
```

The *Completed* value in *STATUS* column is expected.<br>

> oc get pods<br>
NAME                                      READY   STATUS      RESTARTS   AGE
test-pod                                  0/1     Completed   0          4m19s

Logon to NFS server host and verify the SUCCESS file.<br>
> ll /data/nfs2/nfs-storage-test-claim-pvc-2fd9aeaf-4bff-49a9-b6c4-187fe91ce820/
```
-rw-r--r-- 1 root root 0 12-31 15:55 SUCCESS
```
