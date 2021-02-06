# Registry 

https://www.techrepublic.com/article/how-to-set-up-a-local-image-repository-with-podman/


Creating a private image registry is simple and straightforward. Assume that hostname for registry is *thinkde*

# Create registry

Use external storage for registry, here */disk/registry*.

> podman run --privileged -d --name registry -p 5000:5000  -v /disk/registry:/var/lib/registry  registry:2<br>

To allow removing images from the registry, switch on *REGISTRY_STORAGE_DELETE_ENABLED* parameter.<br>

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

# Secure connection

To configure SSL/TLS connection, key and server certificates are necessary. In the case of CA-signed certificate, also a full certificate chain needs to be included. Important: server certificate should be the first, followed by a certificate chain.<br>
Prepare the following data.<br>
* certificate key. For instance: /home/repos/registry-cert/thinkde.sb.com.key.pem
* server certificate including a certificate chain. For instance: /home/repos/registry-cert/cert-chain.pem

Run a long command.<br>

> podman run --privileged -d --name registry -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true 
>   -v /disk/registry:/var/lib/registry \\ <br>
>   -v /home/repos/registry-cert/cert-chain.pem:/certs/cert-chain.pem \\ <br>
>   -v /home/repos/registrycert/thinkde.sb.com.key.pem:/certs/thinkde.sb.com.key.pem \\ <br>
>   -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cert-chain.pem  -e REGISTRY_HTTP_TLS_KEY=/certs/thinkde.sb.com.key.pem registry:2

Verify<br>
> openssl s_client -connect \<hostname\>:5000<br>

Run commands using *https* instead of *http* <br>

> curl -X GET https://\<hostname\>:5000/v2/_catalog<br>
> curl -X GET https://\<hostname\>:5000/v2/db2/tags/list<br>

<br>
If self-signed certificates are used, the registry needs to be included in insecure registry list.<br>
<br>

> vi /etc/containers/registries.conf<br>

```
[registries.insecure]
registries = ['localhost:5000','broth1.fyre.ibm.com:5000']

```
