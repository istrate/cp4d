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
>  oc adm policy add-scc-to-user anyuid -z oracle-sa<br>

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
db19c-oracle-db   1/1     1        
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
> oc set volume dc/db19c-oracle-db --add --claim-name=db19c-oracle-db --claim-size=100G  <br>

Wait until pod is ready.<br>

> oc get pods<br>
```
NAME                               READY   STATUS    RESTARTS   AGE
db19c-oracle-db-5868559fb4-frmc4   1/1     Running   0          45m
```







