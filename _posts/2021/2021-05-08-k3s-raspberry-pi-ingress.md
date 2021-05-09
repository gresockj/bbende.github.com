---
layout: post
title: "K3s on Raspberry Pi - Ingress"
description: ""
category: "Development"
tags: [kubernetes,k3s,raspberry-pi,ingress,traefik]
---
{% include JB/setup %}

In this post we'll look at how ingress works in a K3s cluster. For background, I recommend reading the [Networking Section](https://rancher.com/docs/k3s/latest/en/networking/) of the K3s documentation.

### Ingress Overview

K3s automatically deploys the [Traefik Ingress Controller](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress) and provides a service load balancer called Klipper. To see everything deployed in the `kube-system` namespace, run the following command:

```
kubectl get all --namespace kube-system
```

*NOTE: I have my default context set to `rpi-k3s` so I don't have to specify `--context` on every command.*

This shows the following resources related to Traefik:

```
pod/traefik-97b44b794-dbmz2  
service/traefik
deployment.apps/traefik
replicaset.apps/traefik-97b44b794
```

And the following resource related to the Klipper load balancer:

```
pod/svclb-traefik-fc57n
pod/svclb-traefik-mj4md
pod/svclb-traefik-4qnbh
daemonset.apps/svclb-traefik
```

The traefik deployment contains the specification for a pod with one container using the image `rancher/library-traefik:2.4.8` and having container ports `8000` (web) and `8443` (websecure).   

The traefik service specifies a LoadBalancer for the traffic pod, and maps port `80` of the service to port `8000` on the traefik container, and port `443` of the service to port `8443` on the traefik container.

Klipper then creates a DaemonSet called `svclb-traefik`, which creates a pod on each node to act as a proxy to the service. Each of these pods is accessible from the node's external IP address, and exposes ports `80` and `443`, which map to the respective ports on the service.

The overall setup looks something like this:

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-ingress/01-traefik-klipper-default-setup.png" class="img-responsive">

Next we can deploy an example application and expose it through Traefik.

### Ingress Test

This example is adapted from the Traefik documentation.

Create a namespace called `whoami`:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: whoami
```

Create a deployment for running two pods with the `whoami` container:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  namespace: whoami
  labels:
    app: traefiklabs
    name: whoami

spec:
  replicas: 2
  selector:
    matchLabels:
      app: traefiklabs
      task: whoami
  template:
    metadata:
      labels:
        app: traefiklabs
        task: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
```

Create a service for the `whoami` deployment:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami

spec:
  ports:
    - name: http
      port: 80
  selector:
    app: traefiklabs
    task: whoami
```

Create an ingress to link the `whoami` service to Traefik:

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: whoami
  namespace: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web

spec:
  rules:
    - http:
        paths:
          - path: /bar
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
          - path: /foo
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

With the above ingress we should now be able to access the paths `/foo` or `/bar` on port 80, using the external IP address of any node. The overall setup now looks like the following:

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-ingress/02-whoami-ingress-test.png" class="img-responsive">

If we open a browser and navigate to `http://192.168.1.244/bar`, we get the following output:

```
Hostname: whoami-7d666f84d8-4fs7c
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.4
IP: fe80::c414:76ff:fe4c:75cc
RemoteAddr: 10.42.2.3:39256
GET /bar HTTP/1.1
Host: 192.168.1.244
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:88.0) Gecko/20100101 Firefox/88.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Dnt: 1
Sec-Gpc: 1
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 10.42.0.0
X-Forwarded-Host: 192.168.1.244
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-97b44b794-dbmz2
X-Real-Ip: 10.42.0.0
```

This shows the request reached one of the `whoami` containers at `10.42.1.4`.

If we refresh the page, we now see the response came from the other `whoami` container at `10.42.2.5`.

Since Traefik is performing round-robing load balancing, the requests will continue to alternate between the two `whoami` containers.
