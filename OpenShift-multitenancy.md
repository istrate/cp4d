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
Two users
* *admin* with *admin-cluster* authority 
* limited *developer* having *edit* authority.

It is expected that applications in the project can access services only in the same project and are blocked from accessing services in another project.

# Create projects

As *developer* user.<br>

> oc new-project test<br>
> oc new-project test1<br>
> oc create serviceaccount supersa -n test<br>
> oc create serviceaccount supersa -n test1<br>

As *admin* user<br>
> oc adm policy add-scc-to-user anyuid -z supersa -n test<br>
> oc adm policy add-scc-to-user anyuid -z supersa -n test1<br>

As *developer* user.

> oc new-app --name app --docker-image docker.io/library/nginx:latest -n test<br>
> oc set serviceaccount deployment app supersa -n test<br>
> oc new-app --name app --docker-image docker.io/library/nginx:latest -n test1<br>
> oc set serviceaccount deployment app supersa -n test1<br>



