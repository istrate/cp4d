# Registry 

Creating a private image registry is simple and straightforward. Assume that hostname for registry is *thinkde*

# Create registry

Use external storage for registry, here */disk/registry*.

> podman run --privileged -d --name registry -p 5000:5000  -v /disk/registry:/var/lib/registry  registry:2<br>

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

> podman search thinkde:5000/
```
INDEX          NAME               DESCRIPTION   STARS   OFFICIAL   AUTOMATED
thinkde:5000   thinkde:5000/db2                 0                  
```
# Push/pull

> podman push db2 thinkde:5000/db2<br>

>  podman pull thinkde:5000/db2<br>


