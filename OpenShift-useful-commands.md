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
| oc describe pod/loadtest-74c868c858-45h8z   | grep -A2 -E "Limits|Requests" | Limits for running pod
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
# Misc commands
| Command | Description |
| --- | ---- |
| oc new-project demo --node-selector "tier=special" | Create a project where all containers will be labelled "ties=special"
|  oc new-app --name loadtest  --docker-image quay.io/redhattraining/loadtest:v1.0  --dry-run --save-config -f loadtest.yaml | Creates a yaml specification only