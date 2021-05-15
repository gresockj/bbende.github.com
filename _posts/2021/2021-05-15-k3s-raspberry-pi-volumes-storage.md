---
layout: post
title: "K3s on Raspberry Pi - Volumes & Storage"
description: ""
category: "Development"
tags: [kubernetes,k3s,raspberry-pi]
---
{% include JB/setup %}

In this post we'll look at how volumes and storage work in a K3s cluster.

### Local Path Provisioner

K3s comes with a default [Local Path Provisioner](https://rancher.com/docs/k3s/latest/en/storage) that allows creating a `PersistentVolumeClaim` backed by host-based storage. This means the volume is using storage on the host where the pod is located.  

Let's take a look at an example...

Create a specification for a `PersistentVolumeClaim` and use the `storageClassName` of `local-path`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

Create a specification for a `Pod` that binds to this PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: local-path-test
  namespace: default
spec:
  containers:
  - name: local-path-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: local-path-pvc
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: local-path-pvc
    persistentVolumeClaim:
      claimName: local-path-pvc
```

Create these two resources with `kubectl`:

```
kubectl create -f pvc-local-path.yaml
kubectl create -f pod-local-path-test.yaml
```

After a minute or so, we should be able to see our running pod:

```
kubectl get pods -o wide
NAME              READY   STATUS    RESTARTS   AGE    IP           NODE   
local-path-test   1/1     Running   0          7m7s   10.42.2.34   rpi-2
```

The pod only has one container, so we can get a shell to the container with the following command:

```
kubectl exec -it local-path-test -- sh
/ #
```

Create a test file in `/data`, which is the `mountPath` for the persistent volume:

```
/ # echo "testing" > /data/test.txt
/ # ls -l /data/
total 4
-rw-r--r--    1 root     root             8 May 15 20:24 test.txt
```

The output above showed that this pod is running on the node `rpi-2`.

If we exit out of the shell for the container, and then SSH to the node `rpi-2`, we can expect to the find the file `test.txt` somewhere, since the persistent volume is backed by host-based storage.

```
ssh pi@rpi-2
sudo find / -name test.txt
/var/lib/rancher/k3s/storage/pvc-a5a439ab-ea94-481d-bee2-9e5c0e0f8008_default_local-path-pvc/test.txt
```

This shows that we get out-of-the-box support for persistent storage with K3s, but what if our pod goes down and is relaunched on a different node?

In that case, the new node won't have access to the data from `/var/lib/rancher/k3s/storage` on the previous node. To handle that case, we need distributed block storage.

With distributed block storage, the storage is decouple from the pods, and the `PersistentVolumeClaim` can be mounted to the pod regardless of where the pod is running.

### Longhorn

Longhorn is a "lightweight, reliable and easy-to-use distributed block storage system for Kubernetes." It is also created by Rancher Labs, so it makes the integration with K3s very easy.

For the most part, we can just follow the [Longhorn Install Documentation](https://longhorn.io/docs/0.8.0/install/install-with-kubectl/).

First, make sure you install the `open-iscsi` package on all nodes:

```
sudo apt-get install open-iscsi
```

On my first attempt, I forgot to install this package and ran into issues later.

Deploy Longhorn using `kubectl`:

```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v0.8.0/deploy/longhorn.yaml
```

We can see everything created by listing all resources in the `longhorn-system` namespace:

```
kubectl get all --namespace longhorn-system
```

There are too many resources to list here, but wait until everything is running.

Longhorn has an admin UI that we can access by creating an ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: longhorn-system
  name: longhorn-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: longhorn-ui
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

Originally I wanted to remove the `host` entry to let it bind to all hosts, and specify a more specific path, but currently there is [an issue](https://github.com/longhorn/longhorn/issues/1745) where the path must be `/`.

So we can leave the `host` and `path` as shown, and work around this by creating an `/etc/hosts` entry on our local machine to map the hostname of `longhorn-ui` to one of our nodes:

```
cat /etc/hosts
192.168.1.244 rpi-1 longhorn-ui
192.168.1.245 rpi-2
192.168.1.246 rpi-3
```

Thanks to [Jeric Dy's post](https://www.jericdy.com/blog/installing-k3s-with-longhorn-and-usb-storage-on-raspberry-pi/) for this idea.

In a browser, if you go to `http://longhorn-ui/`, you should see the dashboard like the following:

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-volumes-storage/01-longhorn-ui.png" class="img-responsive">

Now we can do a similar test as we did earlier to verify that everything is working...

Create a specification for a `PersistentVolumeClaim` and use the  `storageClassName` of `longhorn`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

Create a specification for a `Pod` that binds to this PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test
spec:
  containers:
  - name: longhorn-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: longhorn-pvc
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: longhorn-pvc
    persistentVolumeClaim:
      claimName: longhorn-pvc
```

Create these resources with `kubectl`:
```
kubectl create -f pvc-longhorn.yaml
kubectl create -f pod-longhorn-test.yaml
```

After a minute or so, we should be able to see the pod running:

```yaml
kubectl get pods -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP           NODE
longhorn-test   1/1     Running   0          42s   10.42.2.37   rpi-2
```

Get a shell to the container and create a file on the persistent volume:

```
kubectl exec -it longhorn-test -- sh
/ # echo "testing" > /data/test.txt
```

In the Longhorn UI, we should now be able to see this volume under the Volumes tab:

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-volumes-storage/02-longhorn-ui-volumes.png" class="img-responsive">

Drilling in to the PVC shows that the volume is replicated across all three nodes, with the data located under `/var/lib/longhorn/replicas` on each node:

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-volumes-storage/03-longhorn-ui-volume.png" class="img-responsive">

We can't quite as easily see the file on disk because it is stored differently, but we can inspect the location on one of the nodes to see what is there:

```
ssh pi@rpi-2

sudo ls -l /var/lib/longhorn/replicas/
total 4
drwx------ 2 root root 4096 May 15 17:03 pvc-317f4b66-822c-4483-ac44-84541dfac09b-bb6c2f0c

sudo ls -l /var/lib/longhorn/replicas/pvc-317f4b66-822c-4483-ac44-84541dfac09b-bb6c2f0c/
total 49828
-rw------- 1 root root       4096 May 15 17:06 revision.counter
-rw-r--r-- 1 root root 1073741824 May 15 17:06 volume-head-000.img
-rw-r--r-- 1 root root        126 May 15 17:03 volume-head-000.img.meta
-rw-r--r-- 1 root root        142 May 15 17:03 volume.meta
```
In this case, if our pod on node `rpi-2` goes down and is relaunched on another node, it will be able to bind to the same persistent volume, since it came from Longhorn and not from host-based storage.
