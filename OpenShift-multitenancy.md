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

Expose services<br>
> oc expose service app -n test<br>
> oc expose service app -n test1<br>

# Test

Services are accessible from outside, service from *test* can access service *test1* and opposite.<br>

Run<br>

> oc get route -n test<br>
> oc get svc -n test<br>
> oc get route -n test<br>

> oc get pods -n test1<br>
> oc get svc -n test1<br>
> oc get route -n test1<br>

Collect all access data.<br>
| Project | nginx pod | Service Cluster IP | Route
| ---- | ---- | ---- | ---- |
| test | app-68bb6db796-c4t6k | 172.30.128.219 | app-test.apps.boreal.cp.fyre.ibm.com
| test1 | app-68bb6db796-9t9df | 172.30.184.11 | app-test1.apps.boreal.cp.fyre.ibm.com

Verify that *nginx* is accessible externally.<br>
> curl -s app-test.apps.boreal.cp.fyre.ibm.com | grep Thank
```
<p><em>Thank you for using nginx.</em></p>
```

Verify that *nginx* services can access each other internally.<br>
Enter service in *test*
> oc -n test rsh  app-68bb6db796-9t9df  <br>

Call service in project *test1*<br>
> curl -s 172.30.184.11 | grep Thank
```
<p><em>Thank you for using nginx.</em></p>
```


