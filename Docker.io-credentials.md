# Problem
In order to use "BusyBox" image, one has to create an account in *docker.io.* service. https://www.docker.com/. Otherwise, pod creation will be blocked and the following message will be displayed:<br>
```
Failed to pull image "busybox": rpc error: code = Unknown desc = Error reading manifest latest in docker.io/library/busybox: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
# Register docker.io credentials 
Having *docker.io* credentials, the next step is to register them in OpenShift to be used while pulling images from *docker.io*. It is very easy using OpenShift console.<br>
Go to: OpenShift console -> openshift-config project -> Secrets -> pull-secretm -> Actions -> Edit Secret<br>
Go to the bottom, *Add Credentials", fill the form, and it is done.<br>
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-01-17%2019-35-32.png)

# Pull image manually

Another method is to pull *docker.io* image manually and push it to OpenShift internal registry.<br>
https://docs.openshift.com/container-platform/4.5/registry/accessing-the-registry.html<br>
<br>
Sample session.<br>


