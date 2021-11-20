# OpenShift - Jupyter Lab

<br>
(Warning: although the JupyterLab seems to be installed and accessed, I could not create a notebook. It seems that this image cannot work with any arbitrarily assigned user id, there is a permission problem accessing the Jupyter user home directory).<br>
<br>

How to quickly deploy JupyterLab on OpenShift using standard community operator.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-01%2000-36-22.png)

# Create a project and install operator

> oc new-project notebook

From OperatorHub, find and select JupyterLab and install into *notebook* project.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-01%2000-38-53.png)

# Create Jupyter instance

As *notebook* project administrator, go to *Installed operators* tab and install Jupyter instance using the standard configuration.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-01%2000-42-27.png)

Wait until Jupyter instance is deployed.

# Identify Jupyter access token

Using the command line or OpenShift console, display Jupyter pod logs and copy access token.

>  oc logs jupyterlab-sample-64b57fddf8-fvxg6
```
[I 23:40:35.240 LabApp]  or http://127.0.0.1:8888/?token=f807f6cd5535f2ed0760649e4ec63f65f6aa1a718ac72f4b
[I 23:40:35.240 LabApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 23:40:35.244 LabApp]

    To access the notebook, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/nbserver-8-open.html
    Or copy and paste one of these URLs:
        http://jupyterlab-sample-64b57fddf8-fvxg6:8888/?token=f807f6cd5535f2ed0760649e4ec63f65f6aa1a718ac72f4b
     or http://127.0.0.1:8888/?token=f807f6cd5535f2ed0760649e4ec63f65f6aa1a718ac72f4b
[I 22:21:21.115 LabApp] 302 GET / (10.254.8.1) 0.64ms
[I 22:21:21.259 LabApp] 302 GET /lab? (10.254.8.1) 0.59ms
[I 22:22:42.411 LabApp] 302 POST /login?next=%2Flab%3F (10.254.8.1) 0.96ms
```
# Create service NodePort

> oc get deployment
```
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
jupyterlab-operator-controller-manager   1/1     1            1           47h
jupyterlab-sample                        1/1     1            1           47h
```

> oc expose deploy jupyterlab-sample  --port=8888 --type NodePort --name=jupyterlab<br>

Port is assigned by OpenShift, in this example the external port number is 30915

>  oc get svc<br>
```
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jupyterlab          NodePort    172.30.48.246   <none>        8888:30915/TCP   31m
jupyterlab-sample   ClusterIP   172.30.14.21    <none>        8888/TCP         47h
```

# Expose port on HAProxy node

> vi /etc/haproxy/haproxy.cfg
```
frontend jupyter-cluster
        bind *:8888
        default_backend jupyter-cluster
        mode tcp
        option tcplog

backend jupyter-cluster
        balance source
        mode tcp
        server master0 10.22.20.196:30915 check
        server master1 10.22.21.196:30915 check
        server master2 10.22.22.196:30915 check
        server worker0 10.22.23.196:30915 check
        server worker1 10.22.30.195:30915 check
        server worker2 10.22.31.195:30915 check
```
> systemctl reload haproxy

After reconfiguring HAProxy, the *8888* port on HAProxy node is redirected to *8888* Jupyter pod port. Verify that *8888* port is resposive.

> nc -zv /haproxy node/ 8888
```
cat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.46.102.217:8888.
Ncat: 0 bytes sent, 0 bytes received in 0.15 seconds.
```

# Launch JupyterLab

Open browser page: http://{HAProxy node}:8888/lab?token=/access token/

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-06-01%2000-55-43.png)