(Under construction)<br>

# Rook Ceph installation in Open Shift

https://github.com/rook/rook/blob/master/Documentation/ceph-openshift.md<br>

This webpage contains recommendation on Rook Ceph in OpenShit Kubernetes. Below are practical steps on how to do it.

> git clone https://github.com/rook/rook.git<br>
> cd rook/cluster/examples/kubernetes/ceph<br>

Log in to OpenShift cluster using cluster-admin credentials.

> oc login -u admin -p secret<br>


>oc create -f common.yaml<br>

Security objects are created. Also a designed project *rook-ceph* is created.<br>
<br>
Create an operator. Image is pulled from *docker.io*, make sure that *docker.io* credentials are deployed or *rook/ceph:master* image is pulled manually. https://github.com/stanislawbartkowski/CP4D/wiki/Docker.io-credentials<br>
<br>
> oc create -f operator-openshift.yaml<br>

Create all remaining objects.<br>
> oc create -f cluster.yaml<br>
> oc create -f ./csi/rbd/storageclass.yaml<br>
> oc create -f filesystem.yaml<br>
> oc create -f ./csi/cephfs/storageclass.yaml<br>

Verify that all appropriate pods are created. Only "Running" and "Completed" pods should be displayed.<br>

> oc project rook-ceph<br>
```
NAME                                                              READY   STATUS      RESTARTS   AGE
............
csi-cephfsplugin-provisioner-c68f789b8-gwkww                      6/6     Running     0          96s
csi-cephfsplugin-provisioner-c68f789b8-jdqml                      6/6     Running     0          96s
csi-cephfsplugin-v8rbt                                            3/3     Running     0          97s
csi-cephfsplugin-w5jtp                                            3/3     Running     0          97s
csi-rbdplugin-4qllv                                               3/3     Running     0          98s
csi-rbdplugin-provisioner-6c75466c49-wv5x7                        6/6     Running     0          98s
csi-rbdplugin-provisioner-6c75466c49-zmgvr                        6/6     Running     0          98s
csi-rbdplugin-s2ndj                                               3/3     Running     0          98s
csi-rbdplugin-vfj7l                                               3/3     Running     0          98s
csi-rbdplugin-x5lfv                                               3/3     Running     0          98s
csi-rbdplugin-xc8d8                                               3/3     Running     0          98s
rook-ceph-crashcollector-worker0.openshift.cluster.com-fv7pz      1/1     Running     0          18s
rook-ceph-crashcollector-worker1.openshift.cluster.com-6xvzm      1/1     Running     0          94s
...
rook-ceph-osd-prepare-worker0.openshift.cluster.com-zg6pb         0/1     Completed   0          51s
rook-ceph-osd-prepare-worker1..openshift.cluster.com-m547n        0/1     Completed   0          51s
...
```
>  oc create -f toolbox.yaml<br>
<br>
Verify Storage Classes<br>

>oc get sc
```
NAME                            PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block                 rook-ceph.rbd.csi.ceph.com      Delete          Immediate           true                   23h
rook-cephfs                     rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   23h
```

# Test

## Create PVC

> oc create -f csi/rbd/pvc.yaml<br>






