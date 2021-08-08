---
title: Deploying Gitlab CE to a local Kind Kubernetes cluster
author: Benton Snyder
date: 2021-07-31 16:22:00 +0600
categories: [Reference]
tags: [gitlab, kind, kubernetes, helm]
math: false
mermaid: false
---

## Create the Kind cluster

The script below:

- Creates a cluster
- Creates a Registry 
- Exposes 80/443 through the host

Note, only the cluster is necessary. See https://kind.sigs.k8s.io/ for more information. 

create_cluser.sh:

```shell
#!/bin/sh
set -o errexit

# create registry container unless it already exists
reg_name='kind-registry'
reg_port='5000'
running="$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)"
if [ "${running}" != 'true' ]; then
  docker run \
    -d --restart=always -p "127.0.0.1:${reg_port}:5000" --name "${reg_name}" \
    registry:2
fi

# create a cluster with the local registry enabled in containerd
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  # port forward 80 on the host to 80 on this node
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 9000
    hostPort: 5555
    protocol: TCP
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
EOF

# connect the registry to the cluster network
# (the network may already be connected)
docker network connect "kind" "${reg_name}" || true

# Document the local registry
# https://github.com/kubernetes/enhancements/tree/master/keps/sig-cluster-lifecycle/generic/1755-communicating-a-local-registry
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${reg_port}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF
```

Set to executable

```shell
chmod +x create_cluster.sh
```

Execute

```shell
./create_cluster.sh
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
    --set global.smtp.address=smtp.sendgrid.net \
    --set global.smtp.port=465 \
    --set global.smtp.user_name=apikey \
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
