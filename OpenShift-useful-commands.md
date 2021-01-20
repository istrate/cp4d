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
| oc adm policy add-role-to-user edit sbdev -n sb | Grant *edit* privileges for user in a particular project only
| oc adm policy add-role-to-user admin db2admin -n db2 | Making of project admin (db2admin/db2)

## Specify OpenShift credentials
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
| oc debug -t deployment/mysql | Creates ephemeral container similar to specified
| oc debug -t deployment/mysql --image registry.access.redhat.com/ubi8/ubi:8.0 | The same but using a different image
| oc debug -t deployment/mysql --as-root | Run as root, only if the policy allows it

# anyuid ServiceAccount

| Command | Description |
| --- | ---- | 
| oc create sa uuid-sa | Create ServiceAccount
| oc adm policy add-scc-to-user anyuid -z uuid-sa | Assign anyuid
| oc set serviceaccount deployment/appl uuid-sa | Assign ServiceAccount to deployment

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

# Misc commands
| Command | Description |
| --- | ---- |
| oc new-project demo --node-selector "tier=special" | Create a project where all containers will be labelled "ties=special"
| oc new-app --name loadtest  --docker-image quay.io/redhattraining/loadtest:v1.0  --dry-run --save-config -f loadtest.yaml | Creates a yaml specification only
|  oc adm node-logs --role=master -u kubelet | Get master nodes logs
| oc whoami --show-console | OC console hostname