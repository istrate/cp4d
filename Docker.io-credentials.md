# Problem
In order to use "BusyBox" image, one has to create an account in *docker.io.* service. https://www.docker.com/. Otherwise, pod creation will be blocked and the following message will be displayed:<br>
```
Failed to pull image "busybox": rpc error: code = Unknown desc = Error reading manifest latest in docker.io/library/busybox: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
# Register docker.io credentials 
Having *docker.io* credentials, the next step is to register them in OpenShift to be used while pulling images from *docker.io*. It is very easy using OpenShift console.<br>
Go to: OpenShift console -> openshift-config project -> Secrets -> pull-secret -> Actions -> Edit Secret<br>
Go to the bottom, *Add Credentials", fill the form, and it is done.<br>
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-01-17%2019-35-32.png)

# Push image manually

Another method is to pull *docker.io* image manually and push it to OpenShift internal registry.<br>
https://docs.openshift.com/container-platform/4.5/registry/accessing-the-registry.html<br>
<br>
Check accessibility of OpenShift internal registry.<br>
>  nc -zv  image-registry.openshift-image-registry.svc 5000
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 172.30.108.134:5000.
Ncat: 0 bytes sent, 0 bytes received in 0.02 seconds.
```
If not available, log on to any of OpenShift nodes and do the same.<br>

Log in to OpenShift as *admin* user<br>
> oc login https://api.openshift.cluster.com:6443 -u admin -p secret<br>

Log in to *docker.io*<br>
> podman login docker.io<br>

Pull the image to local podman registry<br>
> podman pull docker.io/rook/ceph:master<br>

Retag<br>
> podman tag docker.io/rook/ceph:master  image-registry.openshift-image-registry.svc:5000/openshift/rook/ceph:master<br>

Login to OpenShift internal registry:<br>

> oc login -u kubeadmin -p <password><br>
> podman login -u kubeadmin -p $(oc whoami -t) image-registry.openshift-image-registry.svc:5000<br>

Test.<br>
> podman search  image-registry.openshift-image-registry.svc:5000/
```
INDEX                              NAME                                                                                          DESCRIPTION  STARS   OFFICIAL  AUTOMATED
openshift-image-registry.svc:5000  image-registry.openshift-image-registry.svc:5000/openshift/apicast-gateway
openshift-image-registry.svc:5000  image-registry.openshift-image-registry.svc:5000/openshift/apicurito-ui 
openshift-image-registry.svc:5000  image-registry.openshift-image-registry.svc:5000/openshift/cli
................

````
Push to registry<br>
> podman push  image-registry.openshift-image-registry.svc:5000/openshift/rook/ceph:master<br>