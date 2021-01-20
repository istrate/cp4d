# Rook Ceph installation in Open Shift

https://github.com/rook/rook/blob/master/Documentation/ceph-openshift.md<br>

This webpage contains recommendation on Rook Ceph in OpenShit Kubernetes. Below are practical steps on how to do it.

> git clone https://github.com/rook/rook.git<br>
> cd rook/cluster/examples/kubernetes/ceph<br>

Log in to OpenShift cluster using cluster-admin credentials.

> oc login -u admin -p secret<br>

>oc create -f crds.yaml<br>
>oc create -f common.yaml<br>

Security objects are created. Also a designed project *rook-ceph* is created.<br>
<br>
Create an operator. Image is pulled from *docker.io*, make sure that *docker.io* credentials are deployed or *rook/ceph:master* image is pulled manually. https://github.com/stanislawbartkowski/CP4D/wiki/Docker.io-credentials<br>
<br>
> oc create -f operator-openshift.yaml<br>

> oc project ceph-rook<br>
> oc get pods<br>
```
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-5b9dd84979-fwpbf   1/1     Running   0          48s
```

Create all remaining objects.<br>
> oc create -f cluster.yaml<br>
> oc create -f ./csi/rbd/storageclass.yaml<br>
> oc create -f filesystem.yaml<br>
> oc create -f ./csi/cephfs/storageclass.yaml<br>

Verify that all appropriate pods are created. Only "Running" and "Completed" pods should be displayed.<br>

> oc get pods<br>
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

# Health-check

Open a shell in *rook-ceph-tool* container.<br>
> oc exec -it rook-ceph-tools-7865b9c9f6-7b7bf -- bash<br>
> ceph status<br>
```
  cluster:
    id:     22be353e-57a6-473b-a5f3-4cb73debcf07
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 21m)
    mgr: a(active, since 20m)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 20m), 6 in (since 20m)
 
  data:
    pools:   4 pools, 97 pgs
    objects: 22 objects, 2.2 KiB
    usage:   6.0 GiB used, 2.9 TiB / 2.9 TiB avail
    pgs:     97 active+clean
 
  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr
```
Pay attention to the number of *osd*, number 0 mean that something is wrong.<br>

# Test

## Create PVC

> oc create -f csi/rbd/pvc.yaml<br>
> oc get pvc<br>

Wait until the status is *Bound*.<br>

```
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rbd-pvc   Bound    pvc-f5239735-cd74-4269-86d1-c8b2ffbf9d9d   1Gi        RWO            rook-ceph-block   4s
```
## Write to allocated space
> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/rook-ceph/write-box.yaml<br>
> oc get pods -l app=ceph-test<br>
```
NAME        READY   STATUS      RESTARTS   AGE
write-box   0/1     Completed   0          38s
```
## Read content in allocated space
> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/rook-ceph/read-box.yaml
> oc get pods -l app=ceph-test<br>
```
NAME        READY   STATUS      RESTARTS   AGE
read-box    1/1     Running     0          24s
write-box   0/1     Completed   0          2m21s
```

Open a shell in *read-box* and verify */mnt/SUCCESS* file.<br>
> oc exec -it read-box -- sh<br>
> cat /mnt/SUCCESS<br>
```
Hello world!
```

If successful, delete test objects.<br>
> oc delete pods -l app=ceph-test<br>
> oc delete pvc/rbd-pvc<br>
