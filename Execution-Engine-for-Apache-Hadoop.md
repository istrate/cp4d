# Introduction

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/svc-welcome/hadoopaddon.html
<br>
Execution Engine for Apache Hadoop is one of the services available on IBM Cloud Pak for Data platform. The service integrates CP4D and Hadoop, two Hadoop implementations are supported: CDH (Cloudera) and HDP (former HortonWorks, now Cloudera).
<br>
The installation of the service requires installation and configuration on both sides, Hadoop and CP4D. Steps necessary to activate the service are described here https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/install/install-hadoop.html. <br>
<br>
In the rest of this article, there is a description of practical steps to activate the service.

# Prerequisites

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/install/installing-hadoop-cluster.html

Usually, Hadoop nodes meet the prerequisites required.<br>
In the article, the following hostnames are used.
<br>
| Role | Hostname |
| --- | --- |
| Edge node for Hadoop part | exile1.fyre.ibm.com |
| HDFS Name node | bushily1.fyre.ibm.com |
| CP4D node | https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443	

Hadoop version: HDP

# Install Execution Engine for Apache Hadoop

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/svc/wsj/install-hadoop.html

The installation follows the standard pattern of installing CP4D service. Assembly name: *hadoop-addon*
<br>
> bin/cpd-linux -s repo.yaml -a hadoop-addon --verbose --target-registry-password $(oc whoami -t) --target-registry-username kubeadmin -c managed-nfs-storage --insecure-skip-tls-verify -n zen
<br>
Verify<br>

> bin/cpd-linux status --assembly hadoop-addon --namespace zen<br>

# Install on HDP cluster

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/install/installing-hadoop-cluster.html

## Obtain and install rpm packages

Packages are obtained through PA pages. An examples: *WSPre_Hadoop_Exec_Eng_3011.tar*. Unpack and install:
<br>
> yum install dsxhi-dsx-3.0.1-476.noarch.rpm <br>

Directories after installation.

| Directory | Description |
| ------------ | ---------- |
| /opt/ibm/dsxhi | Binaries and configuration
| /var/log/dsxhi | Log files |

## Configuration 

Collect Hadoop endpoints

| Endpoint | Example |
| ---- | ---- |
| WebHDFS | (here secure) 
| Hive URL | jdbc:hive2://bushily2.fyre.ibm.com:10000
| Ambari URL | http://exile1.fyre.ibm.com:8080 |
| Ambari credentials | admin/admin

> cd /opt/ibm/dsxhi/conf<br>
> cp dsxhi_install.conf.template.HDP  dsxhi_install.conf<br>

Customize *dsxhi_install.conf* property file according to HDP cluster configuration. Below is an example. <br>

> vi dsxhi_install.conf<br>
```
...
# Mandatory - Specify the username and group of the user (dsxhi service user)
# running the dsxhi service.
dsxhi_serviceuser=dsxhi
dsxhi_serviceuser_group=dsxhi
....
# Mandatory - Specify the Ambari URL and admin username for the HDP cluster.
# User will be prompted for password during installation. If the url is not
# specified, some pre-checks for the install will not be performed.
cluster_manager_url=http://exile1.fyre.ibm.com:8080
cluster_admin=admin
...........
# Optional - Specify the hadoop services that dsxhi service should expose.
#exposed_hadoop_services=webhdfs,webhcat,livyspark2,jeg
exposed_hadoop_services=webhdfs,livyspark2,jeg
..........
# Optional - Specify the list of URL for the dsx local clusters that will
# register this dsxhi service. The URL should include the port number if
# necessary.
#known_dsx_list=https://dsxlcluster1.ibm.com,https://dsxlcluster2.ibm.com:31843
known_dsx_list=https://exile1.fyre.ibm.com:31843
...........
# Optional - Provide client side Hive JDBC url:
# (e.g., hive_jdbc_client_url=jdbc:hive2://remotehost:port)
hive_jdbc_client_url=jdbc:hive2://bushily2.fyre.ibm.com:10000
```

## Configure HDFS

Ambari Console->HDFS->Configs->Advanced<br>

Custom core-site<br>

| Property | Value
| ---- | ---- |
| hadoop.proxyuser.dsxhi.groups | * |
| hadoop.proxyuser.dsxhi.hosts | * |
| hadoop.proxyuser.dsxhi.users | * |

Restart all HDP services impacted.

## Install

> cd /opt/ibm/dsxhi/bin<br>
> ./install.py 

## WebHDFS on SSL

If WebHDFS is listening on a secure port, add WebHDFS certificates to Hadoop Engine truststore. It is done automatically by a script.<br>

> cd /opt/ibm/dsxhi/bin<br>
> util/add_cert.sh bushily1.fyre.ibm.com:50470<br>

## Register CP4D URL

> cd /opt/ibm/dsxhi/bin<br>
> ./manage_known_dsx.py -d https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443<br>

Verify and mind Hadoop Engine endpoint.
> ./manage_known_dsx.py -l<br>
```
DSX Local Cluster URL			                                            DSXHI Service URL
https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443			https://exile1.fyre.ibm.com:8443/gateway/zen-cpd-zen
```

After registering CP4D, a corresponding endpoint specification is created.
> ll ../gateway/conf/topologies/zen-cpd-zen.xml 
```
-rw-r--r-- 1 dsxhi dsxhi 2924 11-01 12:44 ../gateway/conf/topologies/zen-cpd-zen.xml

```
## Start/Stop the service
> cd /opt/ibm/dsxhi/bin<br>
> ./start.py<br>
> ./status.py<br>
> ./stop.py<br>

# Register Hadoop Engine in CP4D

CP4D Console->Configure Platform->Hadoop integration->Add Registration<br>
<br>
| Property | Value |
| ---- | ---- |
| Name | (any name) MyCluster |
| Service user ID | dsxhi |
| Service URL | (output from ./manage_known_dsx.py -l command) https://exile1.fyre.ibm.com:8443/gateway/zen-cpd-zen |

# Verify

## Verify HDFS

### Create connection

Add a connection to Watson Studio Project.<br>
Project->Assets->Add to project (top button)->Connection->HDFS via Execution Engine for Hadoop<br>
<br>
WebHDFS URL: *<URL>/webhdfs/v1* <br>
URL: *The Service URL provided while registering Hadoop Cluster*<br>
Example: *https://exile1.fyre.ibm.com:8443/gateway/zen-cpd-zen/webhdfs/v1*
<br>

SSL Certificate<br>
Copy and paste certificate sent by Hadoop Engine gateway<br>
> openssl s_client  -connect exile1.fyre.ibm.com:8443<br>

![](https://github.com/stanislawbartkowski/wikis/blob/master/img/Zrzut%20ekranu%20z%202020-11-02%2023-42-49.png)

### Prepare test data in HDFS
Assuming CP4D logon user is *admin*<br>
> su - hdfs<br>
> hdfs dfs -mkdir /user/admin<br>
> hdfs dfs -chown admin:admin /user/admin<br>
> echo "Hello" >hello.txt<br>
>echo "I am" >>hello.txt<br>
>echo "Stanisław" >>hello.txt<br>
> hdfs dfs -copyFromLocal hello.txt /user/admin<br>
> hdfs dfs -ls /user/admin
```
Found 1 items
-rw-r--r--   3 hdfs admin          6 2020-11-02 22:24 /user/admin/hello.txt
```
### Add dataset in CP4D project

Project->Add to project (top button)->Connected data->Select source<br>

![](https://github.com/stanislawbartkowski/wikis/blob/master/img/Zrzut%20ekranu%20z%202020-11-02%2023-50-30.png)

## Jupyter notebook integrated with remote Hadoop cluster

Watson Studio Jupyter Notebook can run distributed Spark application on remote Hadoop cluster using HDFS/Hadoop resources. The integration is done by JEG (Jupyter Enterprise Gateway), one of CP4D Hadoop Execution Engine services.

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/local/hadoopmodels.html

### Prerequisites

Make sure that JEG service is started and running.
<br>
> cd /opt/ibm/dsxhi/bin<br>
> ./status.py <br>
```
--Status of DSXHI REST server
dsxhi rest is running with PID 11621.
--Status of JEG
JEG is running with PID 11847.
--Status of gateway server
Gateway is running with PID 11909.
```
<br>

_admin_ user cannot start remote JEG session. It is hardcoded in the configuration file.<br>

> vi /opt/ibm/dsxhi/jeg/bin/jeg.sh<br>
```
function appStart {
  export PYTHON_PATH=${VENV_DIR}/lib/python3.6
  export PATH=${VENV_DIR}/bin:$PATH
#  export EG_UNAUTHORIZED_USERS="root,admin,${dsxhi_serviceuser}"
  export EG_UNAUTHORIZED_USERS="root,${dsxhi_serviceuser}"
```
Either remove *admin* from an excluded list (not recommended) or logon to CP4D as a different user (recommended).<br>
<br>
Make sure that SPARK_HOME variable is properly set in kernel specification. It could happen if Spark was installed in HDP after installation of Execution Engine for Hadoop.
> vi /opt/ibm/dsxhi/jeg/kernelspecs/kernels/spark_python_yarn_cluster/kernel.json<br>
<br>

```JSON
{
   "language":"python3",
   "display_name":"Spark - Python (YARN Cluster Mode)",
   "metadata":{
      "process_proxy":{
         "class_name":"enterprise_gateway.services.processproxies.yarn.YarnClusterProcessProxy"
      }
   },
   "env":{
      "SPARK_HOME":"/usr/hdp/current/spark2-client",
      "PYSPARK_PYTHON":"/usr/bin/python",
      "PYTHONPATH":"/opt/ibm/dsxhi/lib/virtualenvs/jeg_base/lib/python3.6/site-packages:/python",
      "SPARK_OPTS":"--master yarn --deploy-mode cluster --name ${KERNEL_ID:-ERROR__NO__KERNEL_ID} --conf spark.sql.catalogImplementation=hive --conf spark.yarn.submit.waitAppCompletion=false --conf spark.yarn.appMasterEnv.PYTHONUSERBASE=.local ${KERNEL_EXTRA_SPARK_OPTS}",
      "LAUNCH_OPTS":"",
      "HADOOP_CONF_DIR":"",
      "EG_IMPERSONATION_ENABLED":"False",
      "EG_KERNEL_LAUNCH_TIMEOUT":"120"
   },
   "argv":[
      "/opt/ibm/dsxhi/jeg/kernelspecs/spark_python_yarn_cluster/bin/run.sh",
      "--RemoteProcessProxy.kernel-id",
      "{kernel_id}",
      "--RemoteProcessProxy.response-address",
      "{response_address}",
      "--RemoteProcessProxy.port-range",
      "{port_range}",
      "--RemoteProcessProxy.spark-context-initialization-mode",
      "lazy"
   ]
}
```
## Create JEG Python 3 environment

Project->Environment-> New environment definition

![](https://github.com/stanislawbartkowski/wikis/blob/master/img/Zrzut%20ekranu%20z%202020-11-03%2013-57-49.png)

## Create a test notebook

Project->Add to project->Notebook->Select runtime (JEG environment)
> !hostname<br>
> rr = spark.sparkContext.textFile("/user/admin/hello.txt")<br>
> print(rr.take(3))<br>

![](https://github.com/stanislawbartkowski/wikis/blob/master/img/Zrzut%20ekranu%20z%202020-11-03%2014-10-03.png)


