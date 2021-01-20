(Under construction)

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-welcome/bigsql.html

# Installation

There are two elements to install:<br>
* IBM DB2 BigSQL in CP4D
* Provision an instance of DB2 BigSQL

# Prerequisites

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_setup_cluster.html

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_install_prereqs.html

Additional remarks:<br>

Make sure that HDFS is accessible from OpenShift cluster nodes. In some environments, HDFS NameNode hostname can resolve as IP in a private network and as different IP ports in a public network used by OpenShift cluster. Assuming that HDFS NameNode as *sawtooth1.fyre.ibm.com*, run the command from any OpenShift nodes:<br>

> nc -zv sawtooth1.fyre.ibm.com 8020<br>
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.30.210.213:8020.
Ncat: 0 bytes sent, 0 bytes received in 0.02 seconds.
```
If there is no connection, it means that HDFS is listening on a private network. The solution is to make HDFS listening on all networks.
https://www.ibm.com/support/pages/ibm-biginsights-how-configure-hadoop-client-port-8020-bind-all-network-interfaces
<br>
* Ambari console -> HDFS -> Configs -> Advanced -> Custom hdfs-site<br>

* Add: dfs.namenode.rpc-bind-host = 0.0.0.0<br>

* Restart HDFS and check NameNode again.<br>

If HDFS is protected by Kerberos, verify that Kerberos server is reachable by OpenShift cluster.

> nc -zv  verse1.fyre.ibm.com 88<br>
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.30.54.109:88.
Ncat: 0 bytes sent, 0 bytes received in 0.04 seconds.
```

# Install IBM DB2 BigSQL

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_install.html

DB2 BigSQL is installed using the same method as all other CP4D services.<br>

> ./cpd-cli adm --repo ./repo.yaml --assembly big-sql --namespace zen --apply<br>
> ./cpd-cli install --repo ./repo.yaml --assembly big-sql   --namespace zen --storageclass managed-zen-storage<br>
> ./cpd-cli status --assembly big-sql --namespace zen
```
Displaying CR status for all assemblies and relevant modules

Status for assembly big-sql and relevant modules in project zen:

Assembly Name                           Status            Version          Arch    
big-sql                                 Ready             7.1.1            x86_64  

  SubAssembly Name                      Status            Version          Arch    
  lite                                  Ready             3.5.1            x86_64  

  Module Name                           Version           Arch             Storage Class     
  big-sql                               7.1.1             x86_64           managed-zen-storage

=========================================================================================
```

# Provision DB2 BigSQL instance

Collect all necessary information.<br>

| Parameter | Example |
| ---- | ---- |
| Ambari URL | http://sawtooth1.fyre.ibm.com:8080 |
| Ambadri admin | admin |
| Ambari password | secret |
| If Hadoop kerberized then Kerberos admin principal. <br> The same used for Hadoop kerberization | hadoopadmin@FYRE.NET
| Kerberos admin password | secret

In a tiny environment, the instance provisioning can be blocked because of lack of resources. The ugly and dirty solution on the fly is to reduce manually the cpu and memory request in Deployment yaml definition. Not recommended in a production environment.<br>

To deploy an instance, go to CP4D console -> Left menu -> Services -> Instances -> New Instance -> DB2 BigSQL tile -> New instance (again) -> Follow user-friendly wizard pages<br>

Review and apply if necessary post-provisioning reconfiguration.<br>
https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_post_prov_tsks.html
# Basic health-check
Check pods up and running, in this example, only one worker node is used.<br>

> oc project zen<br>
> oc get pods | grep bigsql<br>
```
bigsql-1611104274650644-head-7cb969fb7d-vnzsw                1/1     Running     0          9h
bigsql-1611104274650644-worker-0                             1/1     Running     0          9h
bigsql-addon-5f7f6b876f-4rmtm                                1/1     Running     0          9h
bigsql-service-provider-64695f97c9-wsc5w                     1/1     Running     0          9h
```
<br>
Run a command in DB2 BigSQL Head Node pod.<br>
>  oc -it exec bigsql-1611104274650644-head-7cb969fb7d-vnzsw -- bash<br>
> source $HOME/.profile<br>
> bigsql status<br>
```
SERVICE              HOSTNAME                               NODE      PID STATUS

Big SQL Scheduler    head.bigsql-1611104274650644.zen.svc.cluster.local    -    35966 Available
Big SQL Worker       bigsql-1611104274650644-worker-0.bigsql-1611104274650644.zen.svc.cluster.local    1    33759 DB2 Running
Big SQL Master       head.bigsql-1611104274650644.zen.svc.cluster.local    0    42778 DB2 Running
```

> bigsql-admin -health<br>
```
...............
All nodes are registered OK.
------------------------------------------
Processing completed successfully
Total Processing Time (secs) : 26
```

Verify HDFS access.<br>
> hdfs dfs -ls /tmp<br>
```
Found 10 items
drwx------   - ambari-qa hdfs          0 2021-01-10 12:56 /tmp/ambari-qa
drwxr-xr-x   - hdfs      hdfs          0 2021-01-08 14:30 /tmp/entity-file-history
drwx-wx-wx   - hive      hdfs          0 2021-01-20 01:15 /tmp/hive
-rw-r--r--   3 hdfs      hdfs       1024 2021-01-08 14:31 /tmp/id0b0a3020_date310821
...........
```

Verify Hive Server2 connection.<br>
