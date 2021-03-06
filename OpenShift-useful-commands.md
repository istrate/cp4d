# Node labelling

## Commands

| Command | Description |
| --- | ---- |
| oc label node master0.bewigged.os.fyre.ibm.com  tier=gold<br>oc label node master1.bewigged.os.fyre.ibm.com  tier=silver | Assign label to a node|
| oc get nodes -L tier | List nodes and *tier* labels
| oc get nodes --show-labels | List nodes and all labels
| oc label node -l env env- | Remove label from nodes
| oc label namespace network-test name=network-test | Label namespace. Important: use namespace resource, not a project!
## Deploy the application running on a specified node

Create a deployment configuration without creating any object.

>  oc create deployment loadtest --dry-run=client --image quay.io/redhattraining/loadtest:v1.0  -o yaml >loadtest.yaml<br>

```
 spec:
    replicas: 1
    selector:
      matchLabels:
        deployment: loadtest
    nodeSelecter:
        tier: silver
```
> of create -f loadtest.yaml
# Resource limits
## Command
| Command | Description |
| ------ | --------- |
|oc autoscale deployment/loadtest --min 3 --max 40 --cpu-percent 70 | Deployment will scale from 3 to 40 id cpu usage exceeds 70% | 
| oc create quota project-quota --hard cpu="1",memory="2G",configmaps="3",pods="20" | Assign quotas for current project. Add -n if project different then current
| oc describe pod/loadtest-74c868c858-45h8z \| grep -A2 -E "Limits\|Requests" | Limits of running pod
## Limits for deployment
```
   spec:
        containers:
        - image: ' '
          name: loadtest
          ports:
          - containerPort: 8080
            protocol: TCP
          resources:
            requests:
              cpu: "100m"
              memory: "20Mi"

```
# Login and users
## Create OpenShift credentials
| Command | Description
| --- | ---- |
| htpasswd -c -B -b htpasswd admin redhat | Create new htpasswd file and insert admin/redhat credentials
| htpasswd -b htpasswd developer developer | Add credentials to existing htpasswd file
| htpasswd -n -b tester redhat | Generate U/P line only
| oc create secret generic localusers --from-file htpasswd=htpasswd -n openshift-config | Create 
| oc adm policy add-cluster-role-to-user cluster-admin admin | Nominate *admin* user as OpenShift cluster admin
| oc adm policy add-role-to-user edit sbdev -n sb | Grant *edit* privileges for the user in a particular project only
| oc adm policy add-role-to-user admin db2admin -n db2 | Making of project admin (db2admin/db2)
| oc policy add-role-to-group view qa-group | Add role 'view' to group 'qa-group'
| oc get identity | Display the list of current identities
| oc get users  | List of users
| oc adm groups new dev-group | Create new group 'dev-group'
| adm groups add-users dev-group developer| Add user 'developer' to group 'dev-group'
| oc get rolebindings -o wide | Display all role-binding in current project
| oc login --loglevel=10 -u | More verbose


## Specify OpenShift credentials
> oc create secret generic localusers --from-file htpasswd=htpasswd -n openshift-config<br>

> oc get oauth cluster -o yaml >oauth.yaml<br>

Pay attention to *localuser* name the same for 'oauth' and *secret*<br>

```
spec:
  identityProviders:
  - htpasswd:
     fileData:
        name: localusers
    mappingMethod: claim
    name: myusers
    type: HTPasswd    
```

> oc replace -f oauth.yaml<br>

Or edit a configuration directly:<br>

>  oc edit oauth cluster

## Modify existing credentials
Get existing credentials
> oc extract secret/localusers -n openshift-config --confirm<br>

Add new credentials
> htpasswd -b htpasswd manager redhat<br>

Refresh credentials in OpenShift
> oc set data secret/localusers --from-file htpasswd=htpasswd -n openshift-config<br>

## Random password
> MANAGER_PASSWD="$(openssl rand -hex 15)"<br>
> htpasswd -b htpasswd manager ${MANAGER_PASSWD}

## Delete user
| Command | Description 
| ---- | ---- |
| htpasswd -D htpasswd manager | Remove user from htpasswd file
| oc delete user manager | Delete user from OpenShift

## Delete all users
> oc edit oauth<br>
```
spec: {}
```

> oc delete secret localusers -n openshift-config<br>
> oc delete user --all<br>
> oc delete identity --all

# RBAC, role-based access control

| Command | Description
| ---- | ----- |
| oc get scc | List all SCCs (Security Context Contraints)
| oc describe scc anyuid | Details about a single scc
| oc get clusterrolebinding -o wide \| grep -E 'NAME\|self-provisioner' | List all cluster role bindings that reference the self-provisioner cluster role
| oc describe clusterrolebindings self-provisioners | Details about the role, here self-provisioners
| oc adm policy remove-cluster-role-from-group  self-provisioner system:authenticated:oauth | Remove role from virtual group, here the privilege to create a new project
| oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner system:authenticated:oauth | Add role to virtual group
| oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner managers | Add privilege to create a new project to the group managers
| oc adm policy who-can delete user | Verify that user can perform a specific action
| oc get clusterrole | List cluster roles
| oc adm policy who-can use scc privileged | Who can use privileged scc


As a user, cannot create a pod. Run a security *review* to find a solution.<br>

> oc get pod/gitlab-7d67db7875-gcsjl -o yaml  | oc adm policy scc-subject-review -f -<br>
```
RESOURCE                      ALLOWED BY   
Pod/gitlab-65b7d7ffcc-x6gfx   anyuid  
```
As admin, create service account and assign RBAC authority<br>
>  oc create sa gitlab-sa<br>
> oc adm policy add-scc-to-user anyuid -z gitlab-sa<br>

As a user again, assign service account.<br>
> oc set serviceaccount deployment/gitlab gitlab-sa<br>

# Versions, upgrade
| Command | Description
| --- | --- |
| oc patch clusterversion version --type="merge" --patch '{"spec":{"channel":"fast-4.5"}}' | Switch to another channel (fast/stable/candidate)
| oc get clusterversion | Retrieve the cluster version 
| oc get clusterversion -o json | jq ".items[0].spec.channel" | Retrieve the current update channel
| oc adm upgrade | List the available updates
| oc adm upgrade --to-latest=true | Install the latest update 
| oc get clusteroperators | List cluster operators and versions
| oc describe clusterversion | Display more detailed information about cluster version

# Debug

| Command | Description |
| ---- | ---- |
| oc debug -t deployment/mysql | Creates ephemeral container similar to the specified
| oc debug -t deployment/mysql --image registry.access.redhat.com/ubi8/ubi:8.0 | The same but using a different image
| oc debug -t deployment/mysql --as-root | Run as root, only if the policy allows it

# anyuid ServiceAccount

| Command | Description |
| --- | ---- | 
| oc create sa uuid-sa | Create ServiceAccount
| oc adm policy add-scc-to-user anyuid -z uuid-sa | Assign anyuid
| oc set serviceaccount deployment/appl uuid-sa | Assign ServiceAccount to deployment

Deplyment<br>
```
  template:
    metadata:
      labels:
        app: restmock
    spec:
      serviceAccountName: restmock-sa
      containers:
```

# Persistent storage

Assign PVC to the existing PostgreSQL pod. Important: current content will be removed.<br>

> oc set volumes deployment/postgresql-persistent  --add --name postgresql-storage --type pvc --claim-class rook-ceph-block   --claim-mode rwo --claim-size 10Gi --mount-path /var/lib/pgsql --claim-name postgresql-storage<br>


# Pod labels
## Assign labels in pod yaml
```
metadata:
  name: read-box
  labels : {
     app: ceph-test
  }
```
| Command | Description |
| --- | ---- |
| oc get pods --show-labels | Display all labels attached |
| oc get pods -l app=ceph-test | Select pods according to labels |
| oc delete pods -l app=ceph-test | Delete pods according to label |

# Cluster information, health-check
| Command | Description
| ---- | ----- | 
| oc get nodes | List of nodes in the cluster
| oc adm top nodes | Resource consumption by each node
| oc describe node \<node name> | More detailed information about the node
| oc get clusterversion | Retrieves the whole cluster version
| oc describe clusterversion | More detailed information about the cluster
| oc get clusteroperators | Displays current cluster operators, operator managing the cluster
| oc adm node-logs <node name> | All journal logs
| oc adm node-logs -u crio \<node name> | More specific log, you can add --tail \<number\>
| oc adm node-logs -u kubelet \<node name> | More specific log
| oc debug node/\<node name> | Shell session on a specific node
| (inside the node) systemctl status kubelet | Node health-check
| (inside the node) systemctl status cri-o | Node health-check
| (inside the node) journalctl -u kubelet | Detailed log
| (inside node) crictl ps --name openvswitch | Node health-check
| (inside node) crictl images | List images available
| (inside node) crictl pods | List pods
| oc logs \<pod name> | Particular pod logs
| oc logs \<pod-name> -c \<container-name> | Container log, if more then one container in a pod
| oc logs --tail 3  \<pod-name> -c \<container-name> | Container log, in "tail" fashion
| oc debug deployment/\<deployment-name> --as-root | Creates a temporary pod and starts shell session 
| oc rsh \<pod-name> | Open shell session in a pod
| oc cp /local/path \<pod-name>:/container/path | Copy a local file to the container 
| oc port-forward \<pod-name> \<local-port>:\<remote-port> | Creates a TCP tunnel between local machine and pod
| oc get pod --loglevel 6 | More verbose information on 'oc' command
| oc get pod --loglevel 10 | More verbose 
| oc get pod -n </project name> | List pods in a project
| skopeo inspect docker://docker.io/centos/postgresql-12-centos7 | Image info
| skopeo --insecure-policy copy --dest-creds=developer:${TOKEN}  oci:$PWD/ubi-info  docker://image-registry.openshift-image-registry.svc:5000/registry/ubi-info:1.0 --tls-verify=false | Copy image to internal registry
|  podman login -u developer boreal-inf:5000 --tls-verify=false -p ${TOKEN} | Login using token
| oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge | Expose registry route

# Secrets
| Command | Description
| --- | ----- |
|  oc create secret generic mysql   --from-literal user=myuser --from-literal password=redhat123    --from-literal database=test_secrets --from-literal hostname=mysql | Create a secret
| oc new-app --name mysql   --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47 | Will fail because expects environment variables
| oc set env deployment/mysql --from secret/mysql  --prefix MYSQL_ | Assign expected variables taking values from secret
| oc set volume deployment/mysql --add --type secret --mount-path /run/secrets/mysql --secret-name MySQL | Store secrets as text files in mounted directory
| oc new-app --name quotes  --docker-image quay.io/redhattraining/famous-quotes:2.1 | Deploy client application, will fail because environment variables are not set
| oc set env deployment/quotes --from secret/mysql  --prefix QUOTES_ | Assign expected variables from secret
| oc create secret generic quayio --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json --type kubernetes.io/dockerconfigjson | Create external registry temporary credentials valid until API access key expires
| oc secrets link default quayio --for pull | Link to the default service account
| oc secrets link builder quay-registry | Link sa builder to docker quay.io credentials
| oc extract secret/admin-user-details -keys=initial_admin_password --to=- | Extract secret in plain text (dangerous)
# Insecure registry

Assuming a non-secure registry is used to pull docker images. Example, image registry *broth1.fyre.ibm.com:5000* <br>

> oc new-app --name postgresql-persistent --docker-image broth1.fyre.ibm.com:5000/rhel8/postgresql-12 -e POSTGRESQL_USER=redhat -e POSTGRESQL_PASSWORD=redhat123  -e POSTGRESQL_DATABASE=persistentdb --insecure-registry

Flag *--insecure-registry* makes only *oc new-app* completed, pod creation will be blocked by *certificate signed by unknown authority*  error message.<br>

https://docs.openshift.com/container-platform/4.5/post_installation_configuration/preparing-for-users.html#images-configuration-file_post-install-preparing-for-users

The solution is adding registry to list of insecure registries.<br>

> oc edit image.config.openshift.io/cluster<br>
```
spec:
  registrySources:
    insecureRegistries:
    - broth1.fyre.ibm.com:5000

```

After modification, it takes several minutes until the change takes effect because every node is reconfigured. <br>

Wait until all nodes are ready.<br>
> oc get nodes<br>
```
NAME                               STATUS                        ROLES    AGE   VERSION
master0.shrieker.os.fyre.ibm.com   Ready                         master   24d   v1.18.3+fa69cae
master1.shrieker.os.fyre.ibm.com   Ready                         master   24d   v1.18.3+fa69cae
master2.shrieker.os.fyre.ibm.com   NotReady,SchedulingDisabled   master   24d   v1.18.3+fa69cae
worker0.shrieker.os.fyre.ibm.com   Ready                         worker   24d   v1.17.1+40d7dbd
worker1.shrieker.os.fyre.ibm.com   Ready                         worker   24d   v1.17.1+40d7dbd
worker2.shrieker.os.fyre.ibm.com   Ready                         worker   24d   v1.17.1+40d7dbd
worker3.shrieker.os.fyre.ibm.com   Ready,SchedulingDisabled      worker   24d   v1.17.1+40d7dbd
worker4.shrieker.os.fyre.ibm.com   Ready                         worker   24d   v1.17.1+40d7dbd
```
Make sure that on all nodes */etc/container/registries.conf* is updated.<br>

> cat /etc/containers/registries.conf<br>
```
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

[[registry]]
  prefix = ""
  location = "broth1.fyre.ibm.com:5000"
  insecure = true
```
# Limit network access

deny-all.yml<br>

```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-all
spec:
  podSelector: {}

```
allow-specific.yml
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-specific
spec:
  podSelector:
    matchLabels:
      deployment: test
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: network-test
        podSelector:
          matchLabels:
            deployment: sample-app
      ports:
      - port: 8080
        protocol: TCP
```


# Misc commands 
| Command | Description |
| --- | ---- |
| oc new-project demo --node-selector "tier=special" | Create a project where all containers will be labelled "ties=special"
| oc annotate namespace demo openshift.io/node-selector="tier=2" --overwrite | Modify node-selector on existing project
| oc create deployment loadtest --dry-run=client --image quay.io/redhattraining/loadtest:v1.0  -o yaml >loadtest.yaml | Creates a yaml specification only
| oc adm node-logs --role=master -u kubelet | Get master nodes logs
| oc whoami --show-console | OC console hostname
| oc whoami -t | Get authentication token
| TOKEN=$(oc whoami -t) | Store authentication token as env variable
| oc status | Info about the project
| oc get events | Events related to project
| oc describe no | Statistics on all nodes
| oc describe no \| grep Non | Number of running pods on each node
| oc exec -it deployment.apps/postgresql-persistent -- bash | Open pod shell using deployment specification
| oc new-app --name httpd httpd:2.4 | Create httpd pod (ephemeral) 
| oc scale deployment httpd --replicas 3 | Scale application
| oc adm create-bootstrap-project-template -o yaml > /tmp/project-template.yaml | Retrieve project template
| oc expose deployment/loadtest --port 80 --target-port 8080 | Expose deployment, create service
| oc describe dns.operator/default | DNS
| oc describe network/cluster  | Cluster Network Operator
| oc run ubi8 --image=registry.redhat.io/ubi8/ubi --command -- /bin/bash -c 'while true; do sleep 3; done' | Linux box for testing
| oc rollout status deployment tiller | Wait until deployment completed
| oc api-resource | list resources
| skopeo copy docker://quay.io/stanislawbartkowski/php7-original docker-archive:orig.tgz | Copy docker-image as tgz
| oc completion bash >oc_bash_comletion | Prepare
| source oc_bash_comletion | Enable
| oc set env deployment/restapijdbc -e URL="jdbc:postgresql://querydb:5432/querydb" | Set environment variable
| oc set env deployment/restapijdbc --list | List all environment variables and values
# Project template

> oc adm create-bootstrap-project-template -o yaml > /tmp/project-template.yaml <br>

Important: *openshift-config* namespace.<br>

> oc create -f /tmp/project-template.yaml -n openshift-config<br>
> oc edit projects.config.openshift.io/cluster<br>
```
spec:
  projectRequestTemplate:
    name: project-request
```

# Secure
| Command | Description |
| --- | ---- |
| oc extract secrets/router-ca --keys tls.crt -n openshift-ingress-operator | Retrieve router certficates
| tcpdump -i eth0 -A -n port 80 \| grep js | Intercept traffic
| oc create route edge todo-https --service todo-http | Create edge secure route
| curl -I -v  --cacert tls.crt  https://todo-https-network-ingress.apps.jobbery.cp.fyre.ibm.com | Verify certificate
| oc create secret tls todo-certs --cert certs/training.crt --key certs/training.key | Create cert secrets
| oc create route passthrough todo-https  --service todo-https --port 8443 | Create passthrough route

## Prepare a certificate signed by another CA-signed key-certificate

Verify CA-signed key against the password.<br>
> openssl rsa -in training-CA.key -text -passin file:passphrase.txt

Generate a private key.<br>
> openssl genrsa -out training.key 2048 

Generate CSR to be signed
> openssl req -new  -subj "/C=US/ST=North Carolina/L=Raleigh/O=Red Hat CN=todo-https.apps.ocp4.example.com"  -key training.key -out training.csr<br>

* Signed CSR and generate certificate
> openssl x509 -req -in training.csr  -passin file:passphrase.txt  -CA training-CA.pem -CAkey training-CA.key -CAcreateserial   -out training.crt -days 1825 -sha256 -extfile training.ext<br>

* Certificate extension.<br>
```
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
# Replace with your domain name (i.e. domain.example.com)
DNS.1 = *.ocp4.example.com

# Replace with the content of apps.ocp4.example.com
DNS.2 = *.apps.ocp4.example.com

```

# Yaml
| Description | Link |
| -- | --- | 
| Create PostgreSQL | https://github.com/stanislawbartkowski/CP4D/blob/main/yaml/postgresql.yaml
| Create MySQL |??https://github.com/stanislawbartkowski/CP4D/blob/main/yaml/mysql.yaml

# MySQL and WordPress
(as developer)<br>
>  oc create secret generic mysql-secret   --from-literal user=wpuser --from-literal password=redhat123    --from-literal database=wordpress  <br>
> oc new-app --name mysql   --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47<br>

(will fail)
> oc set env deployment/mysql --from secret/mysql-secret  --prefix MYSQL_<br>
> oc new-app --name wordpress --docker-image  docker.io/library/wordpress:5.3.0 --build-env  WORDPRESS_DB_HOST=mysql --build-env WORDPRESS_DB_NAME=wordpress<br>

(will fail)
> oc set env deployment/wordpress --from secret/mysql-secret  --prefix WORDPRESS_DB_<br>

(will fail again)<br>
(as admin)<br>
> oc get pod/wordpress-7f957bfc5b-9kfgt -o yaml    | oc adm policy scc-subject-review -f -
```
RESOURCE                         ALLOWED BY   
Pod/wordpress-7f957bfc5b-9kfgt   anyuid 
```
> oc create sa wordpress-sa<br>
> oc adm policy add-scc-to-user anyuid -z wordpress-sa<br>

(as developer)<br>

> oc set serviceaccount deployment/wordpress wordpress-sa<br>
> oc expose service Wordpress<br>

Logon and make sure that the portal doesn't ask about database access.<br>
# Schedule pods
## Schedule pods on a specific node
> oc new-app --name hello  --docker-image quay.io/redhattraining/hello-world-nginx:v1.0<br>
> oc expose hello<br>
> oc scale --replicas 4 deployment/hello<br>
> oc label node worker0.jobbery.cp.fyre.ibm.com env=dev<br>
> oc label node worker1.jobbery.cp.fyre.ibm.com env=prod<br>
> oc get nodes -L env<br>
> oc edit deployment/hello<br>
```
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      nodeSelector:
        env: dev
      restartPolicy: Always
```
# Create Cockroach DB cluster
## Install operator
As *admin* user, install *Cockroach Operator* operator into *your_project*.
## Create user 
Create *dba* user. Give the user *edit* role in *your_project* and grant the user *view* role in *openshift-operator* project.
## Create ServiceAccount
As *admin* user.

> oc project your_project<br>
> oc create sa cockroach-sa<br>
> oc adm policy add-scc-to-user anyuid -z cockroach-sa

## Create CockroachDB cluster
As *dba* user, navigate to *Create CrdbCluster* page and switch to *Yaml* view.
```
apiVersion: crdb.cockroachlabs.com/v1alpha1
kind: CrdbCluster
metadata:
  name: crdb-tls-example
  namespace: your-project
spec:
  tlsEnabled: true
  nodes: 3
  serviceAccount: cockroach-sa
  dataStore:
    pvc:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        volumeMode: Filesystem

```
Click *Create* and wait until ready.
# Router
* Project *openshift-ingress* -> Pods -> Terminal

> ps -aef <br>
```
UID          PID    PPID  C STIME TTY          TIME CMD
1000560+       1       0  0 Mar18 ?        00:14:28 /usr/bin/openshift-router --v=2
1000560+    1418       1  0 17:16 ?        00:00:05 /usr/sbin/haproxy -f /var/lib/haproxy/conf/haproxy.config -p /var/lib/haproxy/run/haproxy.pid -x /var/lib/haproxy/ru
1000560+    1425       0  0 17:35 pts/0    00:00:00 sh
1000560+    1431    1425  0 17:35 pts/0    00:00:00 ps -aef
```

## Another session to create MySQL instance

> oc create secret generic mysql --from-literal=password=r3dh4t123<br>
> oc new-app --name=mysql --docker-image=registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7<br>
> oc set env deployment/mysql --from=secret/mysql --prefix=MYSQL_ROOT_<br>
> oc get svc<br>
```
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
mysql   ClusterIP   172.30.94.179   <none>        3306/TCP   12m
```

(test) <br>
> oc debug -t deployment/mysql<br>
> mysql -h 172.30.94.179 -u root -p<br>

(mount PVC)
>  oc set volume deployment/mysql --add --name=pvc-mysql --claim-size=2G -t pvc --claim-mode=ReadWriteOnce -m /var/lib/mysql/data<br>
> oc get pvc<br>

(test again as above)<br>

## Service, NodePort

Prepare template.<br>

> oc create service nodeport post-node --dry-run=client --tcp=5432 -o yaml >serv.yml<br>

Important, selector deployment.
```
 selector:
    deployment: postgresl12

```
## Build
| Command | Description
| --- | ---- |
| oc new-app --as-deployment-config --name jhost -i redhat-openjdk18-openshift:1.5 https://github.com/stanislawbartkowski/DO288-apps#manage-builds  --context-dir java-serverhost | New build based on java source code, simple REST service
| oc logs -f bc/jhost | Monitor the progress
| oc get bc | Build config
| oc get build | Builds
| oc start-build bc/jhost | Start new build after committing new changes
| oc cancel-build bc/jhost | Cancel running build

## Internal registry

> oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge<br>
<br>

> HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')<br>

> podman login -u kubeadmin -p $(oc whoami -t) --tls-verify=false $HOST <br>

## RedHat Ubi8 box
> oc new-project redhatbox

Non-root access<br>
> oc run ubi8 --image=registry.redhat.io/ubi8/ubi --command -- /bin/bash -c 'while true; do sleep 3; done'

Root access<br>
> oc create sa redhatsa<br>
> oc adm policy add-scc-to-user anyuid -z redhatsa<br>
> oc run ubi8 --serviceaccount=redhatsa --image=registry.redhat.io/ubi8/ubi --command -- /bin/bash -c 'while true; do sleep 3; done'<br>

Enter the pod.

> oc rsh ubi8<br>

The same for CentOS8

> oc run ce8 --serviceaccount=redhatsa --image=docker.io/library/centos -- /bin/bash -c 'while true; do sleep 3; done'<br>
<br>

> oc run ce8<br>

Create as deployment

The pods started as *oc run* are a not-managed pod, if the pod is deleted, it is not restarted. <br>
In order to have a long-running pod, use *oc new-app*. <br>

> oc new-app --docker-image=docker.io/library/centos  --name=ce8<br>

Immediately after executing this command, the pod will run into *CrashLoopBackOff* cycle. <br>

Apply a command to the container template.<br>
(pay attention to removed - before *image:*)<br>

> oc edit deployment/ce8
```
 spec:
      containers:
      - args:
        - /bin/bash
        - -c
        - while true; do sleep 3; done
        image: docker.io/library/centos@sha256:dbbacecc49b088458781c16f3775f2a2ec7521079034a7ba499c8b0bb7f86875
        imagePullPolicy: IfNotPresent

```

Create a privileged pod<br>
> oc set serviceaccount deployment/ce8 redhatsa<br>

The storage used by the pod is ephemeral, it will disappear every time the pod is recreated. Assign Persistent Volume to the pod and data stored on the persisted volume will survive.<br>

>  oc set volume deployment/ce8 --add --name=ce8   -t pvc --claim-size=20G -m /data<br>

Enter the pod.
> oc get pods<br>
```NAME                   READY   STATUS    RESTARTS   AGE
ce8-7675b66746-sslll   1/1     Running   0          7m43s
```
> oc rsh ce8-7675b66746-sslll<br>

# MSSQL

## Installation

https://developers.redhat.com/blog/2020/10/27/using-microsoft-sql-server-on-red-hat-openshift#deploying_microsoft_sql_server
<br>
Increase the capacity.<br>
> oc new-app --template=mssql2019 -p ACCEPT_EULA=Y -p VOLUME_CAPACITY=20G<br>

Wait until deployment is completed.

## Expose

Create NodePort service.
```YAML
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mssql2019
    app.kubernetes.io/component: mssql2019
    app.kubernetes.io/instance: mssql2019
  name: mssql-ext
spec:
  ports:
  - name: tcp
    port: 1433
    protocol: TCP
    targetPort: 1433
  selector:
    name: mssql
  type: NodePort
```

> oc create -f csv.yaml <br>

>oc get svc<br>
```
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mssql       ClusterIP   172.30.128.147   <none>        1433/TCP         10h
mssql-ext   NodePort    172.30.201.179   <none>        1433:32294/TCP   12m
```

NodePort assigned automatically is *32294*. On HAProxy infrastructure node, reconfigure HAProxy. Example<br>

>vi /etc/haproxy/haproxy.cfg

```
frontend ingress-mssql
        bind *:1433
        default_backend ingress-mssql
        mode tcp
        option tcplog

backend ingress-mssql
        balance source
        mode tcp
        server master0 10.17.43.9:32294 check
        server master1 10.17.46.40:32294 check
        server master2 10.17.48.179:32294 check
        server worker0 10.17.57.166:32294 check
        server worker1 10.17.59.104:32294 check
        server worker2 10.17.61.175:32294 check
```

> systemctl reload haproxy<br>

Verify port responsiveness.
> nc -nv <\haproxy node\> 1433<br>
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.46.197.101:1433.
Ncat: 0 bytes sent, 0 bytes received in 0.22 seconds.

```
## Connect to MSSQL server<br>

Get *SA* password: Workloads -> Secrets -> mssql-secret -> SA_PASSWORD<br>

Download and install mssql tools. 

https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup-tools?view=sql-server-ver15

> sqlcmd -S tcp:\<haproxy node\>,1433 -U SA<br>

```
Password: 
1> 
```
## Create temporary secret from token

> podman login -u \<user\> quay.io<br>
> oc create secret generic quay-registry --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json --type kubernetes.io/dockerconfigjson<br>
> oc secrets link builder quay-registry<br>

## HTTP server

> oc new-app --docker-image=registry.access.redhat.com/rhscl/httpd-24-rhel7  --name=httpd<br>
> oc set volume deployment/httpd --add --name=www  -m //var/www/html  -t pvc --claim-size=20G --claim-class=managed-nfs-storage<br>

> oc expose svc httpd
## Create PVC from command line
```
oc create -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: restapijdbc
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
EOF
```
## ConfigMaps
### Create
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: customers
data:
  key_1: awesomecustomer
  url_1: https://awesome-store.myshopify.com
  key_2: coolcustomer
  url_2: https://your-cool-store.myshopify.com
  key_3: splendidcustomer
  url_3: https://your-splendid-store.myshopify.com
```
### Consume as env variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: read-box
spec:
  containers:
  - name: read-box
    image:  docker.io/busybox
    command: ["sleep","60000"]
    envFrom:
    - configMapRef:
       name: customers
```
### Consume as volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: read-box
spec:
  containers:
  - name: read-box
    image:  docker.io/busybox
    command: ["sleep","60000"]
    volumeMounts:
       - name: customer-volume
         mountPath: /etc/customers
  volumes:
    - name : customer-volume
      configMap:
        name: customers
```
