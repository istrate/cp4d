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

