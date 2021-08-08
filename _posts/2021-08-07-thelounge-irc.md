---
title: Thelounge IRC Client 
author: Benton Snyder
date: 2021-08-07 21:00:00 +0600
categories: [Reference]
tags: [thelounge, irc]
math: false
mermaid: false
---

Create the namespace

```shell
$ kubectl create namespace irc
```

Install the below ingress, service and deployment.

```
##################################################################################################
# Ingress
##################################################################################################
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: thelounge-ingress
  namespace: irc
spec:
  defaultBackend:
    service:
      name: thelounge-service
      port:
        name: http
---
##################################################################################################
# Service
##################################################################################################
kind: Service
apiVersion: v1
metadata:
  name: thelounge-service
  namespace: irc
spec:
  type: LoadBalancer
  selector:
    app: thelounge
  ports:
    - name: http
      port: 80
      nodePort: 30911
      targetPort: 9000
---
##################################################################################################
# Deployment
##################################################################################################
kind: Deployment
apiVersion: apps/v1
metadata:
  name: thelounge-deployment
  namespace: irc
  labels:
    app: thelounge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thelounge
  template:
    metadata:
      labels:
        app: thelounge
    spec:
      containers:
        - name: thelounge
          image: thelounge/thelounge:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9000
          env:
            - name: USERNAME
              value: "test"
          command: ["/bin/bash", "-c"]
          args: ["\
                  sleep 10; \
                  echo Sidecar available; \
                  mkdir -p /var/opt/thelounge/users; \
                  /usr/local/bin/docker-entrypoint.sh; \
                  echo '{\"password\":\"$2a$11$.sR9V89KXUPCB7byuhxL4OywcCP8Z0DsuHI4JsogLvJKJ9TFJ9n5m\",\"log\":true}' > /var/opt/thelounge/users/$(USERNAME).json ; \
                  thelounge start"]
```

Login with test/test. 
