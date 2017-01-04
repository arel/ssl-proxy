# Simple docker & nginx-based ssl-proxy

Protect any HTTP service with HTTPS!
> An Nginx & Docker-based HTTPS/SSL reverse proxy.

## Features

* Fast HTTP2 SSL-enabled reverse proxy
* Advanced CORS Support (w/ credentials, auto hostname)
* Automatic WebSockets Support
* NPN/ALPN Application-Layer Protocol Negotiation [test here](https://tools.keycdn.com/http2-test)
* Great for securing a Docker Registry, Rancher server, Wordpress, etc

> Overview:
>
> 1. [Generate a HTTPS/SSL certificate using letsencrypt.](https://gist.github.com/justsml/63d2884e1cd88d6785999a2eb09cf48e)
> 1. Choose auth method: (Optional)
>     * Simple: define HTTP_USERNAME & HTTP_PASSWORD (recommended)
>     * Htpasswd: Mount an existing passwd file.
> 1. Start an ssl-proxy instance.

### Docker CLI example:

For example, to protect an HTTP service:

1. Requires any working HTTP service (for UPSTREAM_TARGET.) (Supports **local, in-docker, even remote**).
1. Start an instance of `justsml/ssl-proxy:latest` as shown below.

```sh
# Create docker registry - use it's internal exposed port 5000
docker run -d --restart=unless-stopped \
  --name docker-registry \
  -v /data/registry/registry:/var/lib/registry \
  registry:2.5

# Create an ssl-proxy to point at the registry's port 5000 (via UPSTREAM_TARGET option - see below.)
docker run -d --restart=unless-stopped \
  --name ssl-proxy \
  -p 5000:5000 \
  -e 'HTTPS_PORT=5000' \
  -e 'HTTP_USERNAME=devops' \
  -e 'HTTP_PASSWORD=secure' \
  -e 'SERVER_NAME=hub.example.com' \
  -e 'UPSTREAM_TARGET=docker-registry:5000' \
  -e 'SSL_PUBLIC_PATH=/certs/fullchain.pem' \
  -e 'SSL_PRIVATE_PATH=/certs/privkey.pem' \
  -e "ADD_HEADER='Docker-Distribution-Api-Version' 'registry/2.0' always" \
  -v '/certs:/certs:ro' \
  --link 'docker-registry:docker-registry' \
  justsml/ssl-proxy:latest

```

### Example For A Rancher Server

```sh

## Generate SSL Certs using: https://github.com/justsml/system-setup-tools/blob/master/letsencrypt-docker.sh
# Start Rancher w/ local port binding at 8080
docker run -d --restart=unless-stopped \
  --name rancher-server \
  -p 8080 \
  -v /data/rancher/mysql:/var/lib/mysql \
  rancher/server:latest

# Create an ssl-proxy with certs in /certs, (w/o user/pass auth) to point at the local rancher-server's port 8080
docker run -d --restart=unless-stopped \
  --name ssl-proxy \
  -p 8080:8080 \
  -e 'HTTPS_PORT=8080' \
  -e 'SERVER_NAME=rancher.example.com' \
  -e 'UPSTREAM_TARGET=rancher-server:8080' \
  -e 'SSL_PUBLIC_PATH=/certs/fullchain.pem' \
  -e 'SSL_PRIVATE_PATH=/certs/privkey.pem' \
  -v '/certs:/certs:ro' \
  --link 'rancher-server:rancher-server' \
  justsml/ssl-proxy:latest

```



### Docker compose example:

```yaml
version: '2'
services:
  ssl-proxy:
    image: justsml/ssl-proxy:latest
    environment:
    - HTTPS_PORT=8080
    - SERVER_NAME=rancher.example.com
    - UPSTREAM_TARGET=rancher-server:8080
    - SSL_PUBLIC_PATH=/certs/fullchain.pem
    - SSL_PRIVATE_PATH=/certs/privkey.pem
    volumes:
    - /certs:/certs
    links:
    - 'rancher-server:rancher-server'
    ports: [ '8080:8080' ]
  rancher-server:
    image: rancher/server:latest
    expose: [ '8080' ]
    volumes:
    - /data/rancher/mysql:/var/lib/mysql
```




===================
-------------------
===================
-------------------






### Contributing / Dev Notes

> WORK IN PROGRESS:

1. Better CORS support: multi host name
1. haproxy version


```sh
# Publish 'latest' version
docker build -t ssl-proxy:latest .
docker tag ssl-proxy:latest justsml/ssl-proxy:latest
docker push justsml/ssl-proxy:latest
# Push a tagged version:
# docker tag ssl-proxy:latest justsml/ssl-proxy:v1.0.1
# docker push justsml/ssl-proxy:v1.0.1

# Remember to docker pull on servers
docker pull justsml/ssl-proxy:latest

# Local testing:
docker build -t ssl-proxy:latest .
docker rm -f TEST-ssl-proxy
docker run --rm \
  --name TEST-ssl-proxy \
  -v ~/certs/xray:/certs \
  -p 5000:5000 \
  -e 'HTTPS_PORT=5000' \
  -e 'HTTP_USERNAME=devops' \
  -e 'HTTP_PASSWORD=secure' \
  -e 'SERVER_NAME=hub.example.com' \
  -e 'UPSTREAM_TARGET=www.google.com:80' \
  -e 'SSL_PUBLIC_PATH=/certs/fullchain.pem' \
  -e 'SSL_PRIVATE_PATH=/certs/privkey.pem' \
  ssl-proxy:latest


```

