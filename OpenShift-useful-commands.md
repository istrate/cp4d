# Node labelling

## Commands

| Command | Description |
| --- | ---- |
| oc label node master0.bewigged.os.fyre.ibm.com  tier=gold<br>oc label node master1.bewigged.os.fyre.ibm.com  tier=silver | Assign label to a node|
| oc get nodes -L tier | List nodes and *tier* labels
| oc get nodes --show-labels | List nodes and all labels

## Deploy the application running on a specified node
>  oc new-app --name loadtest  --docker-image quay.io/redhattraining/loadtest:v1.0  --dry-run --save-config -f loadtest.yaml
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
|  oc get clusterrolebinding -o wide \| grep -E 'NAME\|self-provisioner' | List all cluster role bindings that reference the self-provisioner cluster role
|  oc describe clusterrolebindings self-provisioners | Details about the role, here self-provisioners
| oc adm policy remove-cluster-role-from-group  self-provisioner system:authenticated:oauth | Remove role from virtual group, here the privilege to create a new project
| oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner system:authenticated:oauth | Add role to virtual group
| oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners self-provisioner managers | Add privilege to create a new project to the group managers

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

# Secrets
| Command | Description
| --- | ----- |
|  oc create secret generic mysql   --from-literal user=myuser --from-literal password=redhat123    --from-literal database=test_secrets --from-literal hostname=mysql | Create a secret
| oc new-app --name mysql   --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47 | Will fail because expects environment variables
| oc set env deployment/mysql --from secret/mysql  --prefix MYSQL_ | Assign expected variables taking values from secret
| oc set volume deployment/mysql --add --type secret --mount-path /run/secrets/mysql --secret-name MySQL | Store secrets as text files in mounted directory
| oc new-app --name quotes  --docker-image quay.io/redhattraining/famous-quotes:2.1 | Deploy client application, will fail because environment variables are not set
| oc set env deployment/quotes --from secret/mysql  --prefix QUOTES_ | Assign expected variables from secret

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

# Misc commands 
| Command | Description |
| --- | ---- |
| oc new-project demo --node-selector "tier=special" | Create a project where all containers will be labelled "ties=special"
| oc new-app --name loadtest  --docker-image quay.io/redhattraining/loadtest:v1.0  --dry-run --save-config -f loadtest.yaml | Creates a yaml specification only
| oc adm node-logs --role=master -u kubelet | Get master nodes logs
| oc whoami --show-console | OC console hostname
| oc whoami -t | Get authentication token
| oc status | Info about the project
| oc get events | Events related to project
| oc describe no | Statistics on all nodes
| oc describe no \| grep Non | Number of running pods on each node
| oc exec -it deployment.apps/postgresql-persistent -- bash | Open pod shell using deployment specification
| oc new-app --name httpd httpd:2.4 | Create httpd pod (ephemeral) 
| oc scale deployment httpd --replicas 3 | Scale application

# Secure
| Command | Description |
| --- | ---- |
| oc extract secrets/router-ca --keys tls.crt -n openshift-ingress-operator | Retrieve router certficates
| tcpdump -i eth0 -A -n port 80 \| grep js | Intercept traffic
| oc create route edge todo-https --service todo-http | Create edge secure route
| curl -I -v  --cacert tls.crt  https://todo-https-network-ingress.apps.jobbery.cp.fyre.ibm.com | Verify certificate

## Prepare a certificate signed by another CA-signed key-certfificate

Verify CA-signed key against the password.<br>
* openssl rsa -in training-CA.key -text -passin file:passphrase.txt

Generate a private key.<br>
* openssl genrsa -out training.key 2048 
Generate CSR to be signed
* openssl req -new  -subj "/C=US/ST=North Carolina/L=Raleigh/O=Red Hat CN=todo-https.apps.ocp4.example.com"  -key training.key -out training.csr<br>
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
| Create MySQL | https://github.com/stanislawbartkowski/CP4D/blob/main/yaml/mysql.yaml

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


