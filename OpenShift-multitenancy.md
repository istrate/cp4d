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
Enter service in *test*.
> oc -n test rsh  app-68bb6db796-c4t6k<br>

Call service in project *test1*<br>
> curl -s 172.30.184.11 | grep Thank
```
<p><em>Thank you for using nginx.</em></p>
```
# Multitenancy
Now we want to harden policy, internal traffic is allowed only between pods in a single project.

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-all
spec:
  podSelector: {}
```
> oc create -n test -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/miltitenancy/deny-all.yaml<br>
> oc create -n test -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/miltitenancy/deny-all.yaml<br>

Test again.<br>
> oc -n test rsh  app-68bb6db796-9t9df<br>
> curl -s 172.30.184.11<br>

Cancel with \<CTRL\>C.

Check traffic inside project *test1*.<br>
> curl -s app-68bb6db796-c4t6k | grep Thank<br>
```
<p><em>Thank you for using nginx.</em></p>
```

Try external access.<br>

> curl -s app-test.apps.boreal.cp.fyre.ibm.com <br>

Break with \<Ctrl\>C. The external access using route is blocked because the *deny-all* policy is very restrictive and does not differentiate between traffic from OpenShift native services like *router* and developer projects.<br>
We need to soften restriction and allow traffic from OpenShift *router* and *monitoring* services.<br>

> oc get project openshift-ingress --show-labels
```
NAME                DISPLAY NAME   STATUS   LABELS
openshift-ingress                  Active   name=openshift-ingress,network.openshift.io/policy-group=ingress,olm.operatorgroup.uid/4e616a48-8931-4b71-8dd4-5c5232a40fa7=,openshift.io/cluster-monitoring=true
```
> oc get project openshift-monitoring --show-labels<br>
```
NAME                   DISPLAY NAME   STATUS   LABELS
openshift-monitoring                  Active   name=openshift-monitoring,network.openshift.io/policy-group=monitoring,olm.operatorgroup.uid/4e616a48-8931-4b71-8dd4-5c5232a40fa7=,openshift.io/cluster-monitoring=true
```

*openshift-ingress* project managing *route* traffic is labelled *network.openshift.io/policy-group=ingress* and *openshift-monitoring* is labelled *network.openshift.io/policy-group=monitoring*

Network policy to release *router* service allows traffic from *project* labelled *network.openshift.io/policy-group: ingress*
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
   name: allow-from-openshift-ingress
spec:
   ingress:
   - from:
     - namespaceSelector:
         matchLabels:
             network.openshift.io/policy-group: ingress
   podSelector: {}
   policyTypes:
   - Ingress
```

>oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/miltitenancy/allow-from-openshift-ingress.yaml -n test<br>
>oc create -f https://raw.githubusercontent.com/stanislawbartkowski/CP4D/main/miltitenancy/allow-from-openshift-ingress.yaml -n test1<br>

> oc create -f https://github.com/stanislawbartkowski/CP4D/raw/main/miltitenancy/allow-from-openshift-monitoring.yaml -n test<br>
> oc create -f https://github.com/stanislawbartkowski/CP4D/raw/main/miltitenancy/allow-from-openshift-monitoring.yaml -n test1<br>

Test again.
>curl -s app-test.apps.boreal.cp.fyre.ibm.com | grep Thank
>curl -s app-test1.apps.boreal.cp.fyre.ibm.com | grep Thank
```
<p><em>Thank you for using nginx.</em></p>

```
