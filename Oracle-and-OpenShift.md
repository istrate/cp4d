https://github.com/oracle/docker-images/tree/main/OracleDatabase/SingleInstance/helm-charts/oracle-db

# Install Helm

https://github.com/helm/helm/releases

# Create Oracle account

https://profile.oracle.com/myprofile/account/create-account.jspx

Make sure that you are authorized to pull Oracle image.<br>

> podman podman login container-registry.oracle.com<br>
> podman pull container-registry.oracle.com/database/enterprise:latest<br>

# Clone Oracle GitHub repository

> git clone https://github.com/oracle/docker-images.git<br>

# Create Helm package

> cd docker-images/OracleDatabase/SingleInstance<br>
>  helm package helm-charts/oracle-db<br>
```
Successfully packaged chart and saved it to: docker-images/OracleDatabase/SingleInstance/oracle-db-1.0.0.tgz
```

# Create OpenShift project and ServiceAccount

> oc new-project oracle<br>
> oc create sa oracle-sa<br>
> oc adm policy add-scc-to-user anyuid -z oracle-sa<br>

# Create Oracle repository credentials.<br>

>  oc create secret docker-registry regcred --docker-server=container-registry.oracle.com --docker-username=\<your-name\> --docker-password=\<your-pword\> --docker-email=\<your-email\>

Add the secret to Oracle Service Account, otherwise, the deployment will fail because of authentication failure.<br>
>  oc secrets link oracle-sa regcred  --for=pull<br>

# Deploy

> helm install db19c oracle-db-1.0.0.tgz

Wait until deployment is visible.<br>

> oc get deployment<br>
```
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
db19c-oracle-db   0/1     0            0           63s
```

Replace the tag in Oracle images as *latest*, not *19.3.0.0* <br>

> oc edit deployment/db19c-oracle-db<br>
```
   - name: ORACLE_EDITION
          value: enterprise
        image: container-registry.oracle.com/database/enterprise:latest
        imagePullPolicy: Always
```

Assign Service Account and PVC
> oc set serviceaccount deployment/db19c-oracle-db oracle-sa<br>
> oc set volume deployment/db19c-oracle-db --add --claim-name=db19c-oracle-db --claim-size=100G <br>

If PVC already exists, delete it and recreate.

> oc get pvc<br>
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
db19c-oracle-db   Bound    pvc-6e44a69c-4c91-4a53-bf95-8e0e2a0394ee   94Gi       RWO            rook-ceph-block   96s
```

Wait until pod is ready.<br>

> oc get pods<br>
```
NAME                               READY   STATUS    RESTARTS   AGE
db19c-oracle-db-5868559fb4-frmc4   1/1     Running   0          45m
```

# Test

Retrieve the admin password. It is exposed in the pod logs.<br>
```
ORACLE EDITION: ENTERPRISE
ORACLE PASSWORD FOR SYS, SYSTEM AND PDBADMIN: u3PHLYyImp
```
The password is also stored in a secret *db19c-oracle-db*.

Enter Oracle pod.<br>
> oc rsh db19c-oracle-db-5868559fb4-n8jvl<br>
> sqlplus / as sysdba<br>

```
SQL*Plus: Release 19.0.0.0.0 - Production on Thu Apr 1 20:29:24 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> show user
USER is "SYS"
SQL> 

```
# Open external traffic to Oracle instance.

## Oracle service NodePort
> oc get svc<br>
```
NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
db19c-oracle-db   NodePort   172.30.147.233   <none>        1521:32753/TCP,5500:30379/TCP   9h
```
The NodePort is *32753*. <br>
## HAProxy service<br>
Modify HAProxy service on infrastructure node.<br>
> vi /etc/haproxy/haproxy.cfg
```
frontend oracle-tcp
        bind *:1521
        default_backend oracle-tcp
        mode tcp
        option tcplog

backend oracle-tcp
        balance source
        mode tcp
        server worker0 <ip address>:32753 check
        server worker1 <ip address>:32753 check
        server worker2 <ip address>:32753 check
```
Restart HAProxy.

> systemctl reload haproxy<br>
## Client node.
Download and install Oracle client software. https://www.oracle.com/technetwork/database/database-technologies/instant-client/downloads/index.html <br>

Connect to Oracle using *sqllplus* utility.<br>

>  sqlplus system@//\<HAProxy hostname\>:1521/ORCLCDB <br>
```
SQL*Plus: Release 19.0.0.0.0 - Production on Thu Apr 1 22:47:58 2021
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Enter password: 
Last Successful login time: Thu Apr 01 2021 18:46:40 +02:00

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> 

```










