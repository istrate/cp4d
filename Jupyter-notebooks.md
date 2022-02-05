# Jupyter

https://github.com/jupyter/docker-stacks/tree/master/examples/openshift

# Simple steps

Create template<br>

> oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/docker-stacks/master/examples/openshift/templates.json<br>

Minimal notebook<br>

> oc new-app --template jupyter-notebook<br>

Different notebook<br>

> oc new-app --template jupyter-notebook  --param NOTEBOOK_IMAGE=jupyter/scipy-notebook:latest <br>

Mind the password<br>
```
     * With parameters:
        * APPLICATION_NAME=notebook
        * NOTEBOOK_IMAGE=jupyter/minimal-notebook:latest
        * NOTEBOOK_PASSWORD=42c3784564713766107db28e5d4c3cc6 # generated

```


Expose NodePort if necessary<br>

> oc expose dc  notebook  --port=8888 --type NodePort --name=jupyter<br>

Enable JupyterLab interface<br>

> oc set env dc/notebook JUPYTER_ENABLE_LAB=true<br>

Keep data on persistent storage<br>

> oc set volume dc/notebook --add --type=pvc --claim-size=1Gi --claim-mode=ReadWriteOnce --claim-name mynotebook-data --name data --mount-path /home/jovyan

# Another method 

## Create service account

> oc create sa uuid-saspark<br>
> oc adm policy add-scc-to-user anyuid -z uuid-saspark<br>

## Create notebook

> oc new-app docker.io/jupyter/all-spark-notebook:latest --name spark  --param  JUPYTER_ENABLE_LAB=true<br>

## Assign service account 

> oc set serviceaccount deployment/spark uuid-saspark

## Expose NodePort

> oc expose deployment  spark  --port=8888 --type NodePort --name=sparkn<br>

> oc get svc<br>
```
spark      ClusterIP   172.30.54.0      <none>        8888/TCP         2m7s
sparkn     NodePort    172.30.205.75    <none>        8888:32676/TCP   9s
```


