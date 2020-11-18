# OpenShift routes

https://docs.openshift.com/container-platform/4.5/networking/routes/route-configuration.html

OpenShift is an extension to Kubernetes service allowing external access to OpenShift/Kubernetes applications. But it requires at least one of the OpenShift nodes to be accessible directly from the client node.<br>

# Architecture

Assume the architecture where the whole OpenShift cluster is running on a private network and the gateway is a separate node running in both networks, private and public, but not being included in the OpenShift cluster

| Node | Network | Role 
| --- | ---- | ---- 
| master0.oc.com | private | Master
| worker0.oc.com | private | Worker
| worker1.oc.com | private | Worker
| worker2.oc.com | private | Worker
| inf.oc.com | private, public | Gateway to the cluster, HAProxy, NFS services
| client.sb.com | public | Client node

The client node can access the Gateway node and run OpenShift console or *oc* command utilizing the HAProxy gateway service.

# 
