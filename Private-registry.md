# Registry 

https://www.techrepublic.com/article/how-to-set-up-a-local-image-repository-with-podman/


Creating a private image registry is simple and straightforward. Assume that hostname for registry is *thinkde*

# Create registry

Use external storage for registry, here */disk/registry*.

> podman run --privileged -d --name registry -p 5000:5000  -v /disk/registry:/var/lib/registry  registry:2<br>

To enable removing from registry.<br>

> podman run --privileged -d --name registry -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true  -v /disk/registry:/var/lib/registry registry:2

# Configure client for HTTP

As a default, *podman* client is reaching registry using a secure connection.

> vi /etc/containers/registries.conf
```
[registries.insecure]
registries = ['thinkde:5000']
```

# Search registry

> curl -X GET http://thinkde:5000/v2/_catalog<br>
```
{"repositories":["db2"]}
```
> curl -X GET http://thinkde:5000/v2/db2/tags/list

> podman search thinkde:5000/
```
INDEX          NAME               DESCRIPTION   STARS   OFFICIAL   AUTOMATED
thinkde:5000   thinkde:5000/db2                 0                  
```
# Push/pull

> podman tag db2 thinkde:5000/db2<br>
> podman push db2 thinkde:5000/db2<br>

>  podman pull thinkde:5000/db2<br>

# skopeo

To maintain the registry, install *skopeo* utility.

> yum install skopeo<br>

Remove from the registry.<br>

>  skopeo delete docker://localhost:5000/mail:latest<br>
