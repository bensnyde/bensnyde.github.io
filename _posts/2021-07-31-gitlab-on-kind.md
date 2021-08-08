---
title: Deploying Gitlab CE to a local Kind Kubernetes cluster
author: Benton Snyder
date: 2021-07-31 16:22:00 +0600
categories: [Reference]
tags: [gitlab, kind, kubernetes, helm]
math: false
mermaid: false
---

Set environment variables:

```shell
export K8S_API_IP="Kubernetes API listen address, must be accessible to Gitlab"
export GITLAB_EXT_IP="Gitlab external IP address"
export GITLAB_DOMAIN="Gitlab base domain"
export GITLAB_ROOT_PASS="Gitlab root user pass"
export GITLAB_ROOT_EMAIL="Gitlab root user email"
export SMTP_PASS="SMTP Pass"
export SMTP_USER="SMTP User"
export SMTP_HOST="SMTP Host"
export SMTP_PORT="SMTP Port"
```

## Create the Kind cluster

Note the Kubernetes API must be accessible to Gitlab to use runners. 

```shell
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "${K8S_API_IP}"
  apiServerPort: 6443
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

Install Helm 

```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Install Gitlab

```shell
kubectl create namespace gitlab

kubectl create secret generic smtp-password \
    -n gitlab \
    --from-literal=password=${SMTP_PASS}
kubectl create secret generic gitlab-gitlab-initial-root-password \
    -n gitlab \
    --from-literal=password=${GITLAB_ROOT_PASS}

helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade --install gitlab gitlab/gitlab \
    --set global.edition=ce \
    --set global.hosts.domain${GITLAB_DOMAIN} \
    --set global.hosts.externalIP=${GITLAB_EXT_IP} \
    --set certmanager-issuer.email=${GITLAB_ROOT_EMAIL} \
    --set postgresql.image.tag=12.7.0 \
    --set global.smtp.enabled=true \
    --set global.smtp.address=${SMTP_HOST} \
    --set global.smtp.port=${SMTP_PORT} \
    --set global.smtp.user_name=${SMTP_USER} \
    --set global.smtp.password.secret=smtp-password \
    --set global.email.from=${GITLAB_ROOT_EMAIL} \
    --set gitlab-runner.runners.privileged=true \
    --set global.hosts.https=true \
    --set global.ingress.enabled=true \
    --set nginx-ingress.controller.hostNetwork=true \
    --set nginx-ingress.controller.kind=DaemonSet \
    --timeout=1h \
    -n gitlab
```
