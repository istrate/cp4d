# Problem
In order to use "BusyBox" image, one has to create an account in *docker.io.* service. https://www.docker.com/. Otherwise, pod creation will be blocked and the following message will be displayed:<br>
```
Failed to pull image "busybox": rpc error: code = Unknown desc = Error reading manifest latest in docker.io/library/busybox: toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
# Register docker.io credentials 
Having *docker.io* credentials, the next step is to register them in OpenShift to be used while pulling images from *docker.io*. It is very easy using OpenShift console.<br>
Go to: OpenShift console -> openshift-config project -> Secrets -> pull-secretm -> Actions -> Edit Secret<br>
Go to the bottom, *Add Credentials", fill the form, and it is done.<br>


