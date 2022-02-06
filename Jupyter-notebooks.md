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

## Assign persistent storage

> oc set volume deployment/spark --add --type=pvc --claim-size=1Gi --claim-mode=ReadWriteOnce --claim-name myspark --name data --mount-path /home/jovyan

## Get token

> oc get pods<br>
```
NAME                     READY   STATUS              RESTARTS   AGE
spark-67f98fbb4b-g7fkt   1/1     Running             0          9m18s
```

> oc logs spark-67f98fbb4b-g7fkt<br>
```
.............
[I 2022-02-05 21:46:07.301 ServerApp] ipyparallel | extension was successfully loaded.
[I 2022-02-05 21:46:07.302 LabApp] JupyterLab extension loaded from /opt/conda/lib/python3.9/site-packages/jupyterlab
[I 2022-02-05 21:46:07.302 LabApp] JupyterLab application directory is /opt/conda/share/jupyter/lab
[I 2022-02-05 21:46:07.307 ServerApp] jupyterlab | extension was successfully loaded.
[I 2022-02-05 21:46:07.307 ServerApp] Serving notebooks from local directory: /home/jovyan
[I 2022-02-05 21:46:07.307 ServerApp] Jupyter Server 1.13.4 is running at:
[I 2022-02-05 21:46:07.307 ServerApp] http://spark-67f98fbb4b-g7fkt:8888/lab?token=22c0e9280508426267c098a4fe9d0eecb2f1b1c964f4494b
[I 2022-02-05 21:46:07.307 ServerApp]  or http://127.0.0.1:8888/lab?token=22c0e9280508426267c098a4fe9d0eecb2f1b1c964f4494b
[I 2022-02-05 21:46:07.307 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 2022-02-05 21:46:07.311 ServerApp] 
    
    To access the server, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/jpserver-8-open.html
    Or copy and paste one of these URLs:
        http://spark-67f98fbb4b-g7fkt:8888/lab?token=22c0e9280508426267c098a4fe9d0eecb2f1b1c964f4494b
     or http://127.0.0.1:8888/lab?token=22c0e9280508426267c098a4fe9d0eecb2f1b1c964f4494b

```
## Run notebook

(assuming proxy node *kist* and NodePort mapped to *8889*)

http://kist:8889/lab?token=22c0e9280508426267c098a4fe9d0eecb2f1b1c964f4494b