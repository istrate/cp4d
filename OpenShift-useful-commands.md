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
# Misc commands
| Command | Description |
| --- | ---- |
| oc new-project demo --node-selector "tier=special" | Create a project where all containers will be labelled "ties=special"
|  oc new-app --name loadtest  --docker-image quay.io/redhattraining/loadtest:v1.0  --dry-run --save-config -f loadtest.yaml | Creates a yaml specification only