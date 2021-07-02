---
layout: post
title: "K3s on Raspberry Pi - Jenkins / Registry (Part 2)"
description: ""
category: "Development"
tags: [kubernetes,k3s,raspberry-pi,jenkins]
---
{% include JB/setup %}

This is second part of setting up a private Docker registry and Jenkins on K3s. The [first part](https://bryanbende.com/development/2021/07/02/k3s-raspberry-pi-jenkins-registry-p1) left off with the private registry being accessible to K3s, and Jenkins being able to execute a basic job through the Kubernetes plugin.

This post will cover setting up a more realistic Jenkins job for an example Spring Boot application, including publishing images to the private registry and using them to deploy the application on K3s.

### Example Application Overview

The code for the example application can be found at [cloud-native-examples/example-app](https://github.com/bbende/cloud-native-examples/tree/main/example-app).

The project is made up of three modules:

* **example-app-api**

  Shared model classes.

* **example-app-backend**

  Spring Boot application with a single REST endpoint for storing/retrieving messages. Messages are stored using a simple in-memory implementation.

* **example-app-frontend**

  Spring Boot application that communicates with the backend and provides a simple UI to create/view messages.

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

### Example Application Images

Since the goal is to deploy the application to K3s, we need a way to build container images for each of the Spring Boot applications.

The `spring-boot-maven-plugin` recently added support for building OCI images through the new `build-image` goal, which utilizes [buildpacks](https://buildpacks.io/) behind the scenes. Unfortunately `buildpacks` does not appear to support `arm`, and I wasn't able to get the `build-image` goal to execute from one of the Raspberry Pis.



### Jenkins Job

### Deployment to K3s
