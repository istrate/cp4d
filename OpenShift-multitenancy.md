https://docs.openshift.com/container-platform/4.7/networking/network_policy/multitenant-network-policy.html

A practical example of OpenShift network tenancy.

# Test scenario

Create two projects/namespaces: *test* and *test1*. The test application is *nginx*.

> oc new-app --name nginx --docker-image docker.io/library/nginx:latest<br>
> oc set serviceaccount deployment nginx \<anyuid enabled Service Account\>

The command to access the *nginx* service.<br>

> curl -s \<ip address\> | grep Thank
```
<p><em>Thank you for using nginx.</em></p>

```

It is expected that applications in the project can access services only in the same project and are blocked from accessing services in another project.


