https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-welcome/bigsql.html

# Installation

There are two elements to install:<br>
* IBM DB2 BigSQL in CP4D
* Provision an instance of DB2 BigSQL

# Prerequisites

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_setup_cluster.html

https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_3.5.0/svc-bigsql/bigsql_install_prereqs.html

Additional remarks<br>

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
Ambari console -> HDFS -> Configs -> Advanced -> Custom hdfs-site<br>

Add: dfs.namenode.rpc-bind-host = 0.0.0.0<br>

Restart HDFS and check NameNode again.<br>

If HDFS is protected by Kerberos, verify that Kerberos server is reachable by OpenShift clusters.

> nc -zv  verse1.fyre.ibm.com 88<br>
```
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Connected to 9.30.54.109:88.
Ncat: 0 bytes sent, 0 bytes received in 0.04 seconds.
```


