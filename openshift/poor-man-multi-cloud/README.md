# The Poor Man's multi cloud

If you don't have enough money to rent resources on two cloud providers, you might be interested by this guide.
It explains how to deploy the multi-cloud demo on an OpenShift cluster in two namespaces and uses an external nginx running on Raspberry PI to simulate geosteering.

## Setup

First, start by creating two namespaces.

```sh
oc new-project quotegame1 --display-name="Quote Game (site 1)"
oc new-project quotegame2 --display-name="Quote Game (site 2)"
```

Deploy the needed operators in those two namespaces.

```sh
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: quotegame1
  namespace: quotegame1
spec:
  targetNamespaces:
  - quotegame1
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: amq-streams
  namespace: quotegame1
spec:
  channel: stable
  name: amq-streams
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: quotegame2
  namespace: quotegame2
spec:
  targetNamespaces:
  - quotegame2
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: amq-streams
  namespace: quotegame2
spec:
  channel: stable
  name: amq-streams
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Using [the ArgoCD instance you deployed earlier](../multi-cloud/argocd/README.md), deploy the application in the two namespaces.

```sh
argocd repo add https://github.com/nmasse-itix/rhforum-2019-quotegame.git
argocd proj create quotegame --description 'Quotegame project'
argocd proj add-source quotegame https://github.com/nmasse-itix/rhforum-2019-quotegame.git
argocd proj add-destination quotegame https://kubernetes.default.svc quotegame1
argocd app create --project quotegame --name quotegame-site1 --repo https://github.com/nmasse-itix/rhforum-2019-quotegame.git --path argocd/multi-cloud/overlays/cluster1 --dest-server https://kubernetes.default.svc --dest-namespace quotegame1 --revision multi-cloud
argocd app create --project quotegame --name quotegame-site2 --repo https://github.com/nmasse-itix/rhforum-2019-quotegame.git --path argocd/multi-cloud/overlays/cluster2 --dest-server https://kubernetes.default.svc --dest-namespace quotegame2 --revision multi-cloud
argocd app sync quotegame-site1
argocd app sync quotegame-site2
```

Then synchronize the CA certificates used by mirror maker.

```sh
oc patch secret -n quotegame1 my-cluster-cluster-ca-cert -p '{"metadata":{"name":"my-cluster-site1-cluster-ca-cert","namespace":"quotegame2"}}' --dry-run -o yaml |oc create -f -
oc patch secret -n quotegame2 my-cluster-cluster-ca-cert -p '{"metadata":{"name":"my-cluster-site2-cluster-ca-cert","namespace":"quotegame1"}}' --dry-run -o yaml |oc create -f -
```

## Install and configure nginx to simulate Geo steering

Recompile nginx as explained [in this article](https://www.looklinux.com/compile-the-nginx-sticky-session-module-in-centos/).
Alternatively, if you are using OpenWRT to host your nginx, there is an [OpenWRT feed](https://github.com/nmasse-itix/openwrt-feeds) ready-to-use!
See [instructions on how to use the OpenWRT SDK](https://www.itix.fr/blog/nginx-with-tls-on-openwrt/).

And use this configuration for your `nginx.conf`:

```
user nginx nginx;
worker_processes 1;

error_log syslog:server=unix:/dev/log,nohostname warn;

events {
    worker_connections 1024;
}

http {
    # don't leak nginx version number in the "Server" HTTP Header
    server_tokens off;

    server_names_hash_bucket_size  64;
    include mime.types;

    log_format main 'HTTP: $remote_addr => $host, request "$request_method $uri" led to $status';
    access_log syslog:server=unix:/dev/log,nohostname main;

    sendfile on;
    keepalive_timeout 65;
    gzip off;

    upstream quotegame {
        # Sticky sessions
        # See https://bitbucket.org/nginx-goodies/nginx-sticky-module-ng/src/master/
        sticky name=srv_id expires=1h domain=quotegame.pi.itix.fr hash=sha1 path=/ secure httponly;

        #
        # HEALTH CHECKS
        #
        # interval: Sets the delay between health checks to 10 seconds
        # fall:     Requires the server to fail three health checks to be
        #           marked as unhealthy
        # rise:     The server must pass two consecutive checks to be
        #           marked as healthy again
        # type:     http: sends a http request packet, receives and parses the
        #           http response to diagnose if the upstream server is alive.
        # timeout:  the health check request's timeout.
        #
        # See https://github.com/yaoweibin/nginx_upstream_check_module
        check interval=3000 fall=3 rise=2 timeout=2000 type=http;

        # List of all backend servers
        server 127.0.0.1:8101;
        server 127.0.0.1:8102;
    }

    server {
        listen 127.0.0.1:8101;
        server_name quotegame;
        access_log off;

        # TODO set the address of the first instance of Quote Game here
        proxy_set_header Host "quotegame-api-quotegame1.apps.ocp4.itix.fr";

        # TODO set the address of the first instance of Quote Game here
        location / {
            proxy_pass http://quotegame-api-quotegame1.apps.ocp4.itix.fr;
        }

        location /api/quote/streaming {
            # TODO set the address of the first instance of Quote Game here
            proxy_pass http://quotegame-api-quotegame1.apps.ocp4.itix.fr;

            # Buffering and cache need to be disabled when the server replies with SSE
            proxy_buffering off;
            proxy_cache off;

            # Here is a "magic trio" making EventSource working through Nginx:
            # https://stackoverflow.com/questions/13672743/eventsource-server-sent-events-through-nginx
            proxy_set_header Connection '';
            proxy_http_version 1.1;
            chunked_transfer_encoding off;
        }
    }

    server {
        listen 127.0.0.1:8102;
        server_name quotegame;
        access_log off;

        # TODO set the address of the second instance of Quote Game here
        proxy_set_header Host "quotegame-api-quotegame2.apps.ocp4.itix.fr";

        location / {
            # TODO set the address of the second instance of Quote Game here
            proxy_pass http://quotegame-api-quotegame2.apps.ocp4.itix.fr;
        }

        location /api/quote/streaming {
            # TODO set the address of the second instance of Quote Game here
            proxy_pass http://quotegame-api-quotegame2.apps.ocp4.itix.fr;

            # Buffering and cache need to be disabled when the server replies with SSE
            proxy_buffering off;
            proxy_cache off;

            # Here is a "magic trio" making EventSource working through Nginx:
            # https://stackoverflow.com/questions/13672743/eventsource-server-sent-events-through-nginx
            proxy_set_header Connection '';
            proxy_http_version 1.1;
            chunked_transfer_encoding off;
        }
    }

    server {
        listen 0.0.0.0:80;
        # TODO: set the server name to your nginx public DNS name
        server_name quotegame.pi.itix.fr;

        proxy_set_header X-Forwarded-For    $remote_addr;
        proxy_set_header X-Forwarded-Proto  "HTTP";

        location / {
            proxy_pass http://quotegame;
        }

        location /api/quote/streaming {
            proxy_pass http://quotegame;

            # Buffering and cache need to be disabled when the server replies with SSE
            proxy_buffering off;
            proxy_cache off;

            # Here is a "magic trio" making EventSource working through Nginx:
            # https://stackoverflow.com/questions/13672743/eventsource-server-sent-events-through-nginx
            proxy_set_header Connection '';
            proxy_http_version 1.1;
            chunked_transfer_encoding off;
        }

        location /status {
            check_status html;
        }
    }
}
```
