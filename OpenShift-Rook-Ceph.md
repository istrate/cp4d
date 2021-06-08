# Rook Ceph installation in Open Shift

https://github.com/rook/rook/blob/master/Documentation/ceph-openshift.md<br>

This webpage contains recommendation on Rook Ceph in OpenShit Kubernetes. Below are practical steps on how to do it.

# Clean disk on worker nodes

If there was a previous storage configuration on worker disks, the Rook Ceph installation  will fail. Clean disk manually or use the following script.

https://github.com/stanislawbartkowski/CP4D/blob/main/rook-ceph/cleanfs.sh

Important: review the script contents before running, the disk cleansing is irreversible!

# Download ceph-rook repository
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

**Important**. In the latest version, a *rook-ceph-default* Service Account is used to create pods. This SA should be granted as *privileged*, otherwise, pods are not allowed to start.<br>

> oc adm policy add-scc-to-user privileged -z rook-ceph-default -n rook-ceph<br>
<br>

> oc create -f operator-openshift.yaml<br>

> oc project rook-ceph<br>
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

Verify that all appropriate pods are created. Only "Running" and "Completed" pods should be displayed. Make sure that *rook-ceph-osd-prepare* pods are present. If they are not, the deployment failed and try to find a cause before proceeding.<br>

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

To make *rook-ceph-block* a default StorageClass, add *default* annotation.<br>
> oc edit sc/rook-ceph-block<br>
```
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  creationTimestamp: "2021-02-18T14:27:05Z"

```
> oc get sc<br>
```
rook-ceph-block (default)   rook-ceph.rbd.csi.ceph.com      Delete          Immediate           true                   9m55s
rook-cephfs                 rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   9m41s
```

# Health-check

Open a shell in *rook-ceph-tool* container.<br>
> oc rsh rook-ceph-tools-7865b9c9f6-7b7bf<br>
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
Pay attention to the number of *osd*, number 0 means that although *ceph* management is running there is no space to be allocated.<br>

Example of the wrong status.<br>
```
 cluster:
    id:     db64f499-d0ea-4ead-9839-d44397037574
    health: HEALTH_WARN
            2 MDSs report slow metadata IOs
            Reduced data availability: 96 pgs inactive
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum a,b,c (age 21m)
    mgr: a(active, since 20m)
    mds: myfs:1 {0=myfs-a=up:creating} 1 up:standby-replay
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   3 pools, 96 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             96 unknown
```

# Troubleshooting<br>
Review logs of completed *rook-ceph-osd-prepare-\<....\>* pods. Example.<br>
```
cephosd: 0 ceph-volume lvm osd devices configured on this node
2021-02-18 12:12:08.820330 W | cephosd: skipping OSD configuration as no devices matched the storage settings for this node "worker4.shrieker.os.fyre.ibm.com"
```
The message means that there are no applicable devices on this node. If it repeats on all nodes, the *ceph* cannot provide any storage for applications.<br>

Further review.<br>
```
021-02-18 12:12:07.884708 I | cephosd: skipping device "vda4" because it contains a filesystem "crypto_LUKS"
2021-02-18 12:12:07.884722 I | cephosd: skipping device "vdb" because it contains a filesystem "LVM2_member"
2021-02-18 12:12:07.884727 I | cephosd: skipping device "vdc" because it contains a filesystem "LVM2_member"
2021-02-18 12:12:07.884866 I | cephosd: configuring osd devices: {"Entries":{}}
```
Devices */dev/vbd* and */dev/vdc* were expected to be included in *ceph* storage. But the devices are labelled as "LVM2_member* and were skipped during devices scanning. 

Solution:<br>
Remove logical volumes and labels attached to devices fit for *ceph*. The devices should be in a "raw" state.<br>
**Warning: be very careful and double-check because the changes are irreversible.**<br>


> lvdisplay<br>
```
--- Logical volume ---
  LV Path                /dev/ceph-d63d273b-20fc-4e27-9d9d-f149fffb80a5/osd-block-cd06d5c4-26c8-44ff-9ef2-eb54435a3f70
  LV Name                osd-block-cd06d5c4-26c8-44ff-9ef2-eb54435a3f70
  VG Name                ceph-d63d273b-20fc-4e27-9d9d-f149fffb80a5
.........
```
> lvremove /dev/ceph-d63d273b-20fc-4e27-9d9d-f149fffb80a5/osd-block-cd06d5c4-26c8-44ff-9ef2-eb54435a3f70<br>

> wipefs /dev/vdb<br>
```
/dev/vdb: 8 bytes were erased at offset 0x00000218 (LVM2_member): 4c 56 4d 32 20 30 30 31
```
> wipefs /dev/vdb -a<br>

> wipefs /dev/vdb<br>
(empty output)
# Test

## Create PVC

> oc new-project test<br>
> oc create -f csi/rbd/pvc.yaml<br>
> oc get pvc<br>

Wait until the status is *Bound*.<br>

```
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
rbd-pvc   Bound    pvc-f5239735-cd74-4269-86d1-c8b2ffbf9d9d   1Gi        RWO            rook-ceph-block   4s
```
## Write to allocated space
>oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/rook-ceph/write-box.yaml<br>
> oc get pods -l app=ceph-test<br>
```
NAME        READY   STATUS      RESTARTS   AGE
write-box   0/1     Completed   0          38s
```
## Read content in allocated space
> oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/rook-ceph/read-box.yaml<br>
> oc get pods -l app=ceph-test<br>
```
NAME        READY   STATUS      RESTARTS   AGE
read-box    1/1     Running     0          24s
write-box   0/1     Completed   0          2m21s
```

Open a shell in *read-box* and verify */mnt/SUCCESS* file.<br>
> oc rsh read-box<br>
> cat /mnt/SUCCESS<br>
```
Hello world!
```

If successful, delete test objects.<br>
> oc delete project test<br>
