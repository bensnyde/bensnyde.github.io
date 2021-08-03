---
title: Deploying Gitlab CE on Minikube
author: Benton Snyder
date: 2021-07-31 16:22:00 +0600
categories: [Reference]
tags: [gitlab, minikube, kubernetes, helm]
math: false
mermaid: false
---

## Installation 

```shell
./minikube-linux-amd64-v1-22-0 start --kubernetes-version=v1.21.2

minikube addons enable ingress

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

## Troubleshooting

Ran into the below issue:

```shell
client.go:230: [debug] Created a new Job called "gitlab-minio-create-buckets-3" in gitlab
upgrade.go:369: [debug] warning: Upgrade "gitlab" failed: failed to create resource: Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1beta1/ingresses?timeout=30s": dial tcp 10.96.8.240:443: connect: connection refused
Error: UPGRADE FAILED: failed to create resource: Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1beta1/ingresses?timeout=30s": dial tcp 10.96.8.240:443: connect: connection refused
helm.go:88: [debug] Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1beta1/ingresses?timeout=30s": dial tcp 10.96.8.240:443: connect: connection refused
failed to create resource
```

Resolved by [deleting the ValidatingWebhookConfiguration](https://stackoverflow.com/a/62044090/1763994)
```
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```
