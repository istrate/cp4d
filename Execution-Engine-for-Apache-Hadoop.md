# Introduction

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/svc-welcome/hadoopaddon.html
<br>
Another installation description step by step: https://developer.ibm.com/recipes/tutorials/i-can-integrate-ibm-cloud-pak-for-data-with-hadoop-for-hdp-cdh/<br>
<br>

Execution Engine for Apache Hadoop is one of the services available on IBM Cloud Pak for Data platform. The service integrates CP4D and Hadoop, two Hadoop implementations are supported: CDH (Cloudera) and HDP (former HortonWorks, now Cloudera).
<br>

An installation of the service requires installation and configuration on both sides, Hadoop and CP4D. Steps necessary to activate the service are described here: https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/install/install-hadoop.html. <br>
<br>
In the rest of this article, there is a description of practical steps to activate the service.

# Prerequisites

In CPD3.x, the Hive access through Execution Engine for Hadoop supports only Cloudera Hadoop distribution. HDP (HortonData Platform) is not supported.<br>

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/install/installing-hadoop-cluster.html

In this article, the following hostnames are used.<br>

<br>

| Role | Hostname |
| --- | --- |
| Edge node for Hadoop part | exile1.fyre.ibm.com |
| HDFS Name node | bushily1.fyre.ibm.com |
| CP4D node | https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443
| Livy URL | http://sawtooth1.fyre.ibm.com:8999

Hadoop version: HDP

# Install Execution Engine for Apache Hadoop

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/svc/wsj/install-hadoop.html

The installation follows the standard pattern of installing CP4D service. Assembly name: *hadoop-addon*
<br>
> bin/cpd-linux -s repo.yaml -a hadoop-addon --verbose --target-registry-password $(oc whoami -t) --target-registry-username kubeadmin -c managed-nfs-storage --insecure-skip-tls-verify -n zen
<br>
Verify<br>

> bin/cpd-linux status --assembly hadoop-addon --namespace zen<br>

# Install HEE component on HDP cluster

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

## Service user

Create *dsxhi* service user and make him the owner of Hadoop Engine files.
> addser dsxhi<br>
> chown -R /opt/ibm/dsxhi<br>

## Configuration 

Collect Hadoop endpoints:<br>

HDP

| Endpoint | Example |
| ---- | ---- |
| Hive URL | jdbc:hive2://bushily2.fyre.ibm.com:10000
| Ambari URL | http://exile1.fyre.ibm.com:8080 |
| Ambari credentials | admin/admin

Cloudera

| Endpoint | Example |
| ---- | ---- |
| Cloudera Manager URL | http://aliquant1.fyre.ibm.com:7180 |
| Cloudera Manager credentials | admin/admin

> cd /opt/ibm/dsxhi/conf<br>
> cp dsxhi_install.conf.template.HDP  dsxhi_install.conf<br>

or (Cloudera)

> cp dsxhi_install.conf.template.CDH  dsxhi_install.conf<br>

Customize *dsxhi_install.conf* property file according to HDP cluster configuration. Below is an example. <br>

As a minimum, it is enough to enable *webhdfs* service.

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
...........
# Optional - Provide client side Hive JDBC url:
# (e.g., hive_jdbc_client_url=jdbc:hive2://remotehost:port)
hive_jdbc_client_url=jdbc:hive2://bushily2.fyre.ibm.com:10000
```

## Configure HDFS

HDP: Ambari Console->HDFS->Configs->Advanced->Custom core-site<br>

Cloudera: Cloudera Manager -> HDFS -> Configuration -> HDFS (Service Wide) -> Cluster-wide Advanced Configuration Snippet (Safety Valve) for core-site.xml

| Property | Value
| ---- | ---- |
| hadoop.proxyuser.dsxhi.groups | * |
| hadoop.proxyuser.dsxhi.hosts | * |
| hadoop.proxyuser.dsxhi.users | * |

Restart all HDP/Cloudera services impacted.

## Install

> cd /opt/ibm/dsxhi/bin<br>
> ./install.py 

## WebHDFS on SSL

If WebHDFS is listening on a secure port, add WebHDFS certificates to Hadoop Engine truststore. It is done automatically by a script.<br>

> cd /opt/ibm/dsxhi/bin<br>
> util/add_cert.sh bushily1.fyre.ibm.com:50470<br>

## Register CP4D URL

> cd /opt/ibm/dsxhi/bin<br>
> ./manage_known_dsx.py -a https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443<br>

Verify CP4D and DSXHI endpoints.<br>
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

CP4D Console->Platform configuration->Systems integration->New integration<br>
<br>
| Property | Value |
| ---- | ---- |
| Name | (any name) MyCluster |
| Service user ID | dsxhi |
| Service URL | (output from ./manage_known_dsx.py -l command) <br> https://exile1.fyre.ibm.com:8443/gateway/zen-cpd-zen |

All runtimes panel:<br>
Push "Jupyter with Python". It can take a number of minutes until completed. Make sure that Status is reported as "Push succeeded".
![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-01-12%2023-22-35.png)

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

Hadoop/HDFS.<br>

>echo "Hello" >hello.txt<br>
>echo "I am" >>hello.txt<br>
>echo "Stanisław" >>hello.txt<br>
> hdfs dfs -copyFromLocal hello.txt /tmp<br>
> hdfs dfs -ls /tmp
```
Found 1 items
-rw-r--r--   3 hdfs admin          6 2020-11-02 22:24 /tmp/hello.txt
```
### Add new platform connection

Cloud Pak for Data -> Connection -> Platform connections -> New connection -> HDFS via Execution Engine for Hadoop

As *WebHDFS URL* copy and paste *WebHDFS* endpoint from Hadoop platform integration panel.<br>
Example: https://internal-nginx-svc:12443/ibm-dsxhi-cdpcluster/webhdfs/v1

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-04%2013-09-12.png)

### Add platform connection to CP4D project

Project -> Add to project -> Connection -> From platform -> HDFSWebHDFSConnection (created as above) -> Create

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-04%2013-20-38.png)

### Add dataset to CP4D project

Project->Add to project (top button)->Connected data->Select source<br>

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-04%2013-22-15.png)

## Jupyter notebook accessing HDFS data

After opening the notebook, Watson Studio can inject a Python code to get access data to remote HDFS file.

Data -> Files -> HelloDataSet -> Insert to code -> Credentials

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-01-15%2023-59-50.png)

Using the data injected, we can create an URL and get access to HDFS using KNOX HDFS Rest/API. https://cwiki.apache.org/confluence/display/KNOX/Examples+WebHDFS

This cell is generated automatically.
```
# @hidden_cell
# The following code contains the credentials for a connection in your Project.
# You might want to remove those credentials before you share your notebook.

from ibm_watson_studio_lib import access_project_or_space
wslib = access_project_or_space()
HelloDataSet_credentials = wslib.get_connected_data("HelloDataSet")h = Hello_txt_credentials
```

Insert Python code
```
h = HelloDataSet_credentials
URL = url = h['url'] + h['datapath']
headers = {'authorization': 'Bearer ' +  h['access_token']}
params = {'op' : 'GETFILESTATUS'}

import requests
r = requests.get(url,headers=headers,params=params)

print(r.content)
Output: b'{"FileStatus":{"accessTime":1610404702071,"blockSize":134217728,"childrenNum":0,"fileId":21279,"group":"admin","length":22,"modificationTime":1610404702276,"owner":"hdfs","pathSuffix":"","permission":"644","replication":3,"storagePolicy":0,"type":"FILE"}}'
```

```
# read HDFS file
params= {'op': 'OPEN'}
r = requests.get(url,headers=headers,params=params)
print(r.text)

Output: 'Hello\nI am\nStanisław\n'
```

## Jupyter notebook integrated with remote Hadoop cluster

Watson Studio Jupyter Notebook can run distributed Spark application on remote Hadoop cluster using HDFS/Hadoop resources. The integration is done by JEG (Jupyter Enterprise Gateway), one of CP4D Hadoop Execution Engine services.

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/wsj/local/hadoopmodels.html

### Endpoints

Make sure that JEG endpoint is defined in CPD4D Hadoop integration panel.

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-07%2023-48-33.png)

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
Cloudera: *SPARK_HOME* looks like:
```
"SPARK_HOME": "/opt/cloudera/parcels/CDH-7.1.5-1.cdh7.1.5.p0.7431829/lib/spark
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

# Livy2

Define *dsxhi* user as *superuser* in Cloudera Livy. <br>

Cloudera Manager -> Livy -> Configuration -> Admin Users/livy.superusers

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-10%2000-17-56.png)

Follow the Livy example:<br>

https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/wsj/local/hadoopmodels.html

Make sure that Livy session is created in Cloudera, it can take several minutes to be ready.<br>

![](https://github.com/stanislawbartkowski/CP4D/blob/main/img/Zrzut%20ekranu%20z%202021-08-10%2000-23-12.png)

# Kerberized HDP

Register *dsxhi* user in Kerberos and obtain appropriate *keytab* file. Modify Hadoop Engine configuration file.<br>

> vi /opt/ibm/dsxhi/conf/dsxhi_install.conf<br>
```
# If the HDP cluster is kerberized, it is mandatory to specify the complete
# path to the keytab for the dsxhi service user and the spnego keytab.
# If the HDP cluster is not kerberized, this field should be left blank.
dsxhi_serviceuser_keytab=/etc/security/keytabs/dsxhi.keytab
dsxhi_spnego_keytab=/etc/security/keytabs/spnego.service.keytab
```

In Cloudera (CDP) it is necessary to activate SPNEGO feature.<br>
https://docs.cloudera.com/cdp-private-cloud-base/7.1.6/security-kerberos-authentication/topics/cm-security-external-authentication-spnego.html<br>
Next step is to identify SPNEGO keytab on edge node. In CDP cluster, all keytabs are stored in */var/run/cloudera-scm-agent/process* directory.<br>

> klist -kt /var/run/cloudera-scm-agent/process/1546338628-hdfs-DATANODE/hdfs.keytab<br>
```
Keytab name: FILE:/var/run/cloudera-scm-agent/process/1546338628-hdfs-DATANODE/hdfs.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 07/25/2021 01:08:57 HTTP/nubbin1.fyre.ibm.com@FYRE.NET
   1 07/25/2021 01:08:57 hdfs/nubbin1.fyre.ibm.com@FYRE.NET

```
This keytab contains SPNEGO (HTTP) principal. Copy it to */home/dsxhi* directory and make it accessible by *dsxhi* user.

> cp  /var/run/cloudera-scm-agent/process/1546338628-hdfs-DATANODE/hdfs.keytab /home/dsxhi/<br>
> chown dsxhi: /home/dsxhi/hdfs.keytab<br>

> vi /opt/ibm/dsxhi/conf/dsxhi_install.conf<br>
```
# If the CDH cluster is kerberized, it is mandatory to specify the complete
# path to the keytab for the dsxhi service user and the spnego keytab.
# If the CDH cluster is not kerberized, these properties should be left blank.
dsxhi_serviceuser_keytab=/home/dsxhi/dsxhi.keytab
dsxhi_spnego_keytab=/home/dsxhi/hdfs.keytab
```

If HDP was Kerberized after Hadoop Engine installation, it is necessary to reinstall it again.<br>
> cd /opt/ibm/dsxhi/bin<br>
> ./manage_known_dsx.py -a https://zen-cpd-zen.apps.rumen.os.fyre.ibm.com:443<br>
> ./uninstall.py<br>
> ./install.py<br>

Also in CP4D, the Hadoop integration should be recreated again.<br>
<br>
Hadoop proxy user *dsxhi* impersonates CP4D user to launch Yarn Spark job. Following steps should be performed for CP4D users:<br>
* register in Kerberos/AD
* create HDFS */user/{CP4D user}* directory and make CP4D user the owner of this directory
* if Ranger is installed, create a policy to authorize CP4D users to submit jobs in Yarn

# CP4D user mapping

CP4D users can be mapped into any other HDP/Cloudera user.<br>

https://knox.apache.org/books/knox-0-12-0/user-guide.html#Default+Identity+Assertion+Provider

Modif Hadoop Engine gateway configuration file:<br>
> vi /opt/ibm/dsxhi/gateway/conf/topologies/zen-cpd-zen.xml
```
..........
  <provider>
         <role>identity-assertion</role>
         <name>Default</name>
         <enabled>true</enabled>
        <param>
            <name>principal.mapping</name>
            <value>admin=guest;</value>
        </param>
      </provider>
.........
```
Restart the gateway:<br>
> cd /opt/ibm/dsxhi/bin<br>
> ./stop.py<br>
> ./start.py<br>

The CP4D *admin* user will reach the Hadoop cluster as *guest* user. All prerequisites regarding Hadoop users (Kerberos registering, HDFS home directory and Yarn authorization) should be applied to *guest* user as well.<br>