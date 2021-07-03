---
layout: post
title: "K3s on Raspberry Pi - Jenkins / Registry (Part 2)"
description: ""
category: "Development"
tags: [kubernetes,k3s,raspberry-pi,jenkins]
---
{% include JB/setup %}

This is the second part of setting up Jenkins and a private Docker registry on K3s. The [first part](https://bryanbende.com/development/2021/07/02/k3s-raspberry-pi-jenkins-registry-p1) left off with the private registry up and running and accessible to K3s, and Jenkins being able to execute a basic job through the Kubernetes plugin.

This post will cover setting up a more realistic Jenkins job for an example Spring Boot application, including publishing images to the private registry and running them on K3s.

### Example Application Overview

The code for the example application can be found at [cloud-native-examples/example-app](https://github.com/bbende/cloud-native-examples/tree/main/example-app).

The project is made up of three modules:

* **example-app-api** - Shared model classes

* **example-app-backend** - Spring Boot application with a single REST endpoint for storing/retrieving messages, messages are stored using a simple in-memory implementation

* **example-app-frontend** - Spring Boot application that communicates with the backend and provides a simple UI to create/view messages

The frontend `application.properties` contains the location of the backend application:

```
example.app.backend.server.protocol=http
example.app.backend.server.host=localhost
example.app.backend.server.port=8081
example.app.backend.server.messages-context=/messages
```

These values would potentially need to be overridden depending how the application is being deployed.

If we start each Spring Boot application locally from Intellij, we should see the following...

*Backend*
<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-jenkins-pt2/01-example-app-backend-intellij.png" class="img-responsive">

*Frontend*
<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-jenkins-pt2/02-example-app-frontend-intellij.png" class="img-responsive">

*Browser at http://localhost:8080*
<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-jenkins-pt2/03-example-app-frontend-ui-local.png" class="img-responsive">

### Building Images

Since our goal is to deploy the application to K3s, we need a way to produce container images for the backend and frontend Spring Boot applications.

The `spring-boot-maven-plugin` recently added support for building OCI images through the new `build-image` goal. This relies on [buildpacks](https://buildpacks.io/) behind the scenes, which unfortunately does not support `arm`, so the `build-image` goal won't succeed from one of the Raspberry Pis.

Another popular option is [Jib](https://github.com/GoogleContainerTools/jib), a library from Google for containerizing Java applications. Jib has plugins for Maven and Gradle, and has the ability to build images without a local Docker daemon. This is particularly useful for our Jenkins job which is going to execute in a pod that won't have access to a Docker daemon.

As a side note, if you do need to access a Docker daemon from within a container, a common approach seems to be generally referred to as "Docker in Docker (DIND)". This boils down to mounting `/var/run/docker.sock` from the host into the container, so the container is actually using the host's Docker daemon. Since K3s uses `containerd` as the runtime, there isn't a Docker daemon on the host anyway.

The configuration of the Jib plugin is done in [pluginManagement at the example-app level](https://github.com/bbende/cloud-native-examples/blob/02-jib/example-app/pom.xml#L45-L98) so that it can be shared across the frontend and backend applications.

There are three profiles available:

* **jib-docker-daemon** - Builds to the local Docker daemon

* **jib-docker-hub** - Builds and pushes to Docker Hub, requires running `docker login` first

* **jib-k3s-private** - Builds and pushes to the private registry in K3s, requires specifying username/password of the registry with `-Djib.registry.username` and `-Djib.registry.password`

We can test building the images with our local Docker daemon by executing:

```
mvn clean package -Pjib-docker-daemon
```

After a successful build, running the command `docker images` should show the following:

```
REPOSITORY         TAG              IMAGE ID       CREATED          SIZE
example-backend    0.0.1-SNAPSHOT   ea3241a5cfd6   25 seconds ago   261MB
example-backend    latest           ea3241a5cfd6   25 seconds ago   261MB
example-frontend   0.0.1-SNAPSHOT   1cf33e4a9207   37 seconds ago   265MB
example-frontend   latest           1cf33e4a9207   37 seconds ago   265MB
```

Next we can setup a Jenkins job that will use the `jib-k3s-private` profile to build and publish the images to private registry.

### Jenkins Job

First, we need to create a Jenkins Credential to store the username/password for the private registry so that these values can be referenced securely from the build configuration.

Navigate to *Manage Jenkins* -> *Manage Credentials* -> *"Jenkins" Store* -> *Domain "Global Credentials (unrestricted)"*, and then click  *Add Credential*.

<img src="{{ BASE_PATH }}/assets/images/k3s-rpi-jenkins-pt2/04-jenkins-docker-credentials.png" class="img-responsive">

Enter the username and password for the registry, and enter a unique id for the credential, we will call it `docker-registry-private`. The id can be anything, we just need to reference it later.

Now we can create the new pipeline. From the main Jenkins page, click *New Item* and add a pipeline named `cloud-native-examples`. Under the *Pipeline Definition*, select `Pipeline Script` and enter the following script:

```
pipeline {
    environment {
        GIT_REPO_URL = 'https://github.com/bbende/cloud-native-examples.git'
        GIT_REPO_BRANCH = 'main'
        REGISTRY_URL = 'docker-registry-service.docker-registry.svc.cluster.local:5000'
        REGISTRY_CREDENTIAL = credentials('docker-registry-private')
    }
    agent {
        kubernetes {
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: maven
      image: maven:3.8.1-openjdk-11
      command: ["tail", "-f", "/dev/null"]
      imagePullPolicy: IfNotPresent
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "1Gi"
      volumeMounts:
            - name: jenkins-maven
              mountPath: /root/.m2
  volumes:
    - name: jenkins-maven
      persistentVolumeClaim:
        claimName: jenkins-maven-pvc
"""
        }
    }
    stages {
        stage('Git Clone') {
            steps {
                git(url: "${GIT_REPO_URL}", branch: "${GIT_REPO_BRANCH}")
            }
        }
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn package -Pjib-k3s-private -Djib.registry.username=$REGISTRY_CREDENTIAL_USR -Djib.registry.password=$REGISTRY_CREDENTIAL_PSW -DsendCredentialsOverHttp=true'
                }
            }
        }
    }
}
```

Let's breakdown each part of the pipeline...

* **environment** - Defines variables for use in other parts of the pipeline, including a variable for the private registry credential from calling `credentials('docker-registry-private')`

* **agent/kubernetes** - Defines an agent for executing the pipeline in Kubernetes with a field for the YAML definition of the agent Pod which uses the image `maven:3.8.1-openjdk-11`

* **stage('Git Clone')** - Clones the git repository for the project

* **stage('Build')** - Executes the Maven build using the `jib-k3s-private` profile and overrides variables to specify the username/password for the registry using the credential environment variables

As an optimization, the Maven container mounts a persistent volume to `/root/.m2` which comes from a Longhorn PVC. This allows the local Maven repository to be persisted across builds, instead of downloading every dependency on every build.

The `REGISTRY_URL` is defined as `docker-registry-service.docker-registry.svc.cluster.local`. This is because we have a service named `docker-registry-service` in the `docker-registry` namespace, but we are accessing it from Jenkins in the `jenkins` namespace, so we need the fully qualified hostname.

The registry is secured with username/password authentication, but the internal service at `docker-registry-service` is not TLS-enabled, so we have to add `-DsendCredentialsOverHttp=true` to allow Jib to authenticate over http. This is acceptable for internal communication on our example cluster, but is not recommended for a real environment.  

After creating the pipeline we can click *Build Now* to start a build. From watching the Console Output of the build, we should see something like the following:

```
Created Pod: k3s jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Scheduled] Successfully assigned jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3 to rpi-3
Still waiting to schedule task
‘cloud-native-examples-26-k493m-qzrwp-3xpv3’ is offline
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][SuccessfulAttachVolume] AttachVolume.Attach succeeded for volume "pvc-af0ab285-fe51-48a8-be4c-f106bea22566"
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Pulled] Container image "maven:3.8.1-openjdk-11" already present on machine
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Created] Created container maven
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Started] Started container maven
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Pulled] Container image "pi4k8s/inbound-agent:4.3" already present on machine
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Created] Created container jnlp
[Normal][jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3][Started] Started container jnlp
```

This shows that the pod `jenkins/cloud-native-examples-26-k493m-qzrwp-3xpv3` was launched to execute the build. The persistent volume for the Maven repo was then attached and the containers were created and started.

The set of containers is a combination of the default Pod template created in Part 1, and the YAML definition in the pipeline. So the combined set of containers includes `maven:3.8.1-openjdk-11`, `pi4k8s/inbound-agent:4.3`, and `jnlp`.

The rest of the build should continue with a standard Maven build of a Spring Boot application. At some point in the build output, we should see the frontend and backend images being published:

```
[INFO Built and pushed image as docker-registry-service.docker-registry.svc.cluster.local:5000/example-frontend, docker-registry-service.docker-registry.svc.cluster.local:5000/example-frontend:0.0.1-SNAPSHOT-k3s
...
[INFO Built and pushed image as docker-registry-service.docker-registry.svc.cluster.local:5000/example-backend, docker-registry-service.docker-registry.svc.cluster.local:5000/example-backend:0.0.1-SNAPSHOT-k3s
```

We can verify that the images are now available in the private registry by running the following command from our laptop:

```
curl -k -X GET --basic -u registry https://docker.registry.private/v2/_catalog | python -m json.tool
```

This shows the following output:

```
{
    "repositories": [
        "arm32v7/nginx",
        "example-backend",
        "example-frontend"
    ]
}
```

### Testing Images on K3s

Now that the images are available in the private registry, we can test deploying pods on K3s to verify the images work correctly:

```
kubectl create namespace example-app

kubectl run example-backend --image docker.registry.private/example-backend --namespace example-app

kubectl run example-frontend --image docker.registry.private/example-frontend --namespace example-app
```

The images are prefixed with the registry location of `docker.registry.private`, which must line up with the configuration in `/etc/rancher/k3s/registries.yaml`.

If we check the status of the pods in the `example-app` namespace, we should see two pods come up running:

```
kubectl get pods --namespace example-app
NAME               READY   STATUS    RESTARTS   AGE
example-backend    1/1     Running   0          102s
example-frontend   1/1     Running   0          59s
```

This proves K3s is able to pull images from the private registry and the images were correctly built for `arm64`.

There would be a bit more work to correctly deploy the application. Currently the frontend isn't correctly configured to access the backend, and the frontend isn't accessible from outside the cluster, but those are topics for another post.
