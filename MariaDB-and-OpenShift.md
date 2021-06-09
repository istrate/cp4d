# Practical steps on how to deploy MariaDB to OpenShift cluster

## Docker

Make sure you have access to *registry.redhat.io* registry. <br>

Simple test, if fails try another docker registry.<br>

> podman pull registry.redhat.io/rhscl/mariadb-105-rhel7
```
Trying to pull registry.redhat.io/rhscl/mariadb-105-rhel7:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 9b0c218cbfb1 done  
Copying blob f8c518873786 done  
Copying blob 93156a512b98 [=>------------------------------------] 3.6MiB / 72.9MiB
Copying blob d133eb51475d [--------------------------------------] 711.1KiB / 60.6MiB
```
## Create user and project

As OpenShift *admin* user, create *mariadb* admin user.<br>

> htpasswd -nb mariadb secret<br>

Modify *OAuth* secret and verify *mariadb* credentials.<br>

> oc login -u mariadb -p secret<br>

As *admin* user.

> oc new-project mariadb<br>
> oc adm policy add-role-to-user admin mariadb -n mariadb<br>

## Create MariaDB instance

As *mariadb* user in *mariadb* project.<br>

> oc login -u mariadb -p secret<br>

> oc new-app --docker-image=registry.redhat.io/rhscl/mariadb-105-rhel7  --name=mariadb  -e MYSQL_ROOT_PASSWORD=secret<br>

Assign Persistent Volume.<br>

> oc set volume deployment/mariadb --add  --claim-name=mariadb  --mount-path=/var/lib/mysql/data  -t pvc --claim-size=10G  

If necessary, create *NodePort* for external access.<br>

> oc expose deployment/mariadb --name mariadbn --type=NodePort

