---
title: Deploying Gitlab CE on Minikube
author: Benton Snyder
date: 2021-07-31 16:22:00 +0600
categories: [Reference]
tags: [gitlab, minikube, kubernetes, helm]
math: false
mermaid: false
---

```
kubectl create namespace gitlab
kubectl create secret generic smtp-password --from-literal=password=[password] -n gitlab
kubectl create secret generic gitlab-initial-root-password --from-literal=password=[password] -n gitlab

helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade --install gitlab gitlab/gitlab \
    --set global.edition=ce \
    --set global.hosts.domain=[domain] \
    --set global.hosts.externalIP=[external_ip] \
    --set certmanager.install=false,global.ingress.configureCertmanager=false \
    --set nginx-ingress.enabled=false,global.ingress.class="nginx" \
    --set gitlab.gitlab-shell.service.type=NodePort \
    --set gitlab.gitlab-shell.service.nodePort=32695 \
    --set postgresql.image.tag=12.7.0 \
    --set gitlab.migrations.initialRootPassword.secret=gitlab-initial-root-password \
    --set global.smtp.enabled=true \
    --set global.smtp.address=[smtp_host] \
    --set global.smtp.port=[smtp_port] \
    --set global.smtp.user_name=apikey \
    --set global.smtp.password.secret=smtp-password \
    --set global.email.from=[email_adddress] \
    --set gitlab-runner.runners.privileged=true \
    --timeout=20m \
    -n gitlab \
    --debug
```
