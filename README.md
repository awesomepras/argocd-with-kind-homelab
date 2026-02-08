# Local Kubernetes Cluster with ArgoCD, MetalLB, and Observability

This repository documents a local multi-node Kubernetes cluster using KinD (Kubernetes in Docker) on Ubuntu 24.04, with ArgoCD, MetalLB, and optional observability tooling.


---
{#top}
## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Tooling Installation](#tooling-installation)
- [Install KinD](#install-kind)
- [Install kubectl](#install-kubectl)
- [Container Runtime](#container-runtime)
- [KinD Cluster Setup](#kind-cluster-setup)
- [MetalLB Configuration](#metallb-configuration)
- [kube-proxy Configuration (strictARP)](#kube-proxy-configuration-strictarp)
- [Install ArgoCD](#install-argocd)
- [Configure ArgoCD](#configure-argocd)
- [Expose ArgoCD via LoadBalancer](#expose-argocd-via-loadbalancer)
- [Verification](#verification)
- [Observability (Optional)](#observability-optional)
- [Podman Troubleshooting (Deprecated)](#podman-troubleshooting-deprecated)
- [Upcoming Enhancements](#upcoming-enhancements)
- [Troubleshooting](#troubleshooting)

---

## Overview

This setup includes:

- Multi-node KinD cluster
- Docker Engine as container runtime
- MetalLB for LoadBalancer services
- ArgoCD installed via Helm
- Ubuntu firewall configuration (iptables + UFW)
- Optional Prometheus, Grafana, and Loki

---

## Prerequisites
---

## Tooling Installation

### Install Go

```bash
sudo snap install go --classic
export PATH=$PATH:$HOME/go/bin
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### Install KinD

```bash
go install sigs.k8s.io/kind@v0.31.0
which kind
```

### Install kubectl

Install kubectl using the official Kubernetes documentation:

https://kubernetes.io/docs/tasks/tools/

[↑ Back to top](#top)

---

## Container Runtime

### Recommended: Docker Engine

Docker Engine is the recommended runtime for KinD in this setup.

Reasons:

- Predictable networking
- Reliable MetalLB behavior
- No rootless or privileged port workarounds
- Easier host-to-cluster traffic forwarding

Install Docker Engine using official documentation:

https://docs.docker.com/engine/install/ubuntu/

Podman is documented only for reference as i could not get it to work completely. See the Podman troubleshooting section below.

[↑ Back to top](#top)

---

## KinD Cluster Setup

Clone the repository containing the KinD configuration:

```bash
git clone https://github.com/initcron/k8s-code.git
cd k8s-code/helper/kind/
```

Create a three-node cluster:

```bash
kind create cluster \
  --config kind-three-node-cluster.yaml \
  --name bootstrap
```
***Note: cluster name - bootstrap (instead of default - kind)***

Verify cluster status:

```bash
kubectl cluster-info --context kind-bootstrap
```
[↑ Back to top](#top)

---

## MetalLB Configuration

MetalLB enables Service type LoadBalancer in a local KinD cluster.
Refer to scripted install: https://github.com/awesomepras/kind-local-cluster/blob/main/k3d/2-install-metallb.sh

More information: https://github.com/awesomepras/kind-local-cluster/blob/main/k3d/README.md


### Enable IP Forwarding on Host

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Identify Docker Network Subnet

```bash
docker network inspect kind | grep Subnet
```

Example output:

```text
"Subnet": "172.18.0.0/16"
```

### Configure MetalLB Address Pool

Create or update layer2-config.yaml:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: static-pool # make sure this matches aht name in bgpadvertisement
  namespace: metallb-system
spec:
  addresses:
    - 172.18.255.200-172.18.255.250
  autoAssign: true
```

Apply MetalLB manifests and configuration following official documentation:

https://metallb.universe.tf/configuration/

[↑ Back to top](#top)

---

## kube-proxy Configuration (strictARP)

MetalLB Layer 2 mode requires strictARP enabled.

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed 's/strictARP: false/strictARP: true/' | \
  kubectl apply -f - -n kube-system
```

Restart kube-proxy:

```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

Verify MetalLB speaker pods:

```bash
kubectl get pods -n metallb-system -l component=speaker -o wide
kubectl logs -n metallb-system -l component=speaker
```

[↑ Back to top](#top)

---

## Install ArgoCD

Create namespace and install using Helm:

```bash
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd
```

### Optional: Pin ArgoCD Version
Find the latest version of argoCD Chart
``` helm search repo argo/argo-cd --versions   |head -3```

values.yaml:

```yaml
argo-cd:
  server:
    image:
      repository: quay.io/argoproj/argocd
      tag: v3.2.6 #latest
```

Upgrade release:

```bash
helm upgrade argocd argo/argo-cd -n argocd -f values.yaml
```

---
## Configure ArgoCD

After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

(You should delete the initial secret afterwards as suggested by the [Getting Started Guide:](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)  

```bash
kubectl get secret argocd-initial-admin-secret -n argocd
kubectl delete secret argocd-initial-admin-secret -n argocd
```

If you’ve forgotten the password: Reset it by deleting the admin.password and admin.passwordMtime entries from the argocd-secret secret:

```bash
kubectl patch secret argocd-secret -n argocd -p '{"data": {"admin.password": null, "admin.passwordMtime": null}}'
```

Then restart the ArgoCD server:

```bash
kubectl rollout restart deployment.apps/argocd-server -n argocd
```

[↑ Back to top](#top)
---

## Expose ArgoCD via LoadBalancer

Patch the ArgoCD server service:

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

Wait for external IP:

```bash
kubectl get svc argocd-server -n argocd -w
```
[↑ Back to top](#top)


---

## Verification

Retrieve ArgoCD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Access the UI:

- URL: https://EXTERNAL_IP
- Username: admin
- Password: retrieved above

[↑ Back to top](#top)

---

## Observability (Optional)

### Prometheus and Grafana

```bash
helm install prom prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

Grafana admin password:

```bash
kubectl get secret -n monitoring prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Loki

In-cluster push endpoint:

```text
http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

Port-forward for testing:

```bash
kubectl port-forward -n monitoring svc/loki-gateway 3100:80
```
[↑ Back to top](#top)

---

## Podman Troubleshooting (Deprecated)

Podman is not recommended for this setup. I couldnt get argoCD running with podman.  

Known issues:

- systemd delegation requirements
- privileged port restrictions
- networking isolation

### systemd Delegation Fix

```bash
sudo mkdir -p /etc/systemd/system/user@.service.d
sudo tee /etc/systemd/system/user@.service.d/delegate.conf <<EOF
[Service]
Delegate=yes
EOF
sudo systemctl daemon-reload
```

### Allow Privileged Ports

```bash
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Create Cluster with Podman

```bash
systemd-run --scope --user -p Delegate=yes \
  kind create cluster \
  --config kind-three-node-cluster.yaml \
  --name bootstrap
```

[↑ Back to top](#top)

---

## Upcoming Enhancements

Planned work:

- ArgoCD installation using Kustomize
- App-of-apps GitOps pattern
- GCP PoC Toolkit integration
- Multi-cluster ArgoCD fleet management
- Terraform-based bootstrap

---

## Troubleshooting

| Issue | Resolution |
|------|------------|
| No LoadBalancer IP | Verify MetalLB speakers and strictARP |
| ArgoCD UI unreachable | Check iptables and UFW routing |
| Cluster unreachable after reboot | Check Docker service status |
| Ports below 1024 blocked | Use NodePort >= 1024 |

---

Tested on Ubuntu 24.04.
[↑ Back to top](#top)


**References:**  
https://kubernetes-tutorial.schoolofdevops.com/kind_create_cluster/  
https://www.arthurkoziel.com/setting-up-argocd-with-helm/  
https://piotrminkowski.com/2022/06/28/manage-kubernetes-cluster-with-terraform-and-argo-cd/  
https://github.com/argoproj/argoproj-deployments/tree/master/argocd  
https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/  
https://github.com/GoogleCloudPlatform/gke-poc-toolkit/tree/main/demos/fleets  
https://github.com/GoogleCloudPlatform/gke-poc-toolkit-demos/tree/main/gke-fleets-with-argocd  


