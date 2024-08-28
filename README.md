# Argocd KSOPS

This repository contains the K3D setup for the ArgoCD and includes kustomize with sops encryption setup.

## Prerequisites

- Install Docker

```bash
sudo apt install docker.io
sudo groupadd docker
sudo usermod -aG docker $USER
```

- Install K3D

```bash
curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

- Install Kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Start k3d cluster

Using 2 worker nodes and 1 control/master node

```bash
sudo k3d cluster create argocd-ksops --api-port 6550 --agents 2 --servers 1
```

## Get the kubeconfig file

```bash
sudo k3d kubeconfig get argocd-ksops > ~/.kube/config
```

## Setup GPG key

```bash
export GPG_NAME="argocd-ksops.cluster"
export GPG_COMMENT="argocd ksops gpg"
gpg --batch --gen-key gen-key.conf
```

List GPG keys

```bash
gpg --list-secret-keys argocd-ksops.cluster
export GPG_ID=<YOUR GPGP SEC ID OUPUT>
```

Export the public key

```bash
gpg --export-secret-keys --armor "${GPG_ID}" > gpg.key
gpg --armor --export "${GPG_ID}" > gpg.pub
gpg --delete-secret-keys "${GPG_ID}"
```

## Install ArgoCD

Apply the secret for KSOPS decryption to the cluster:

```bash
kubectl create namespace argocd
kubectl create secret generic sops-gpg \
--namespace=argocd \
--from-file=gpg.key
```

Apply the local with Kustomize:

```bash
kubectl apply -k argocd/overlays/local/
```

### Get/Update password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login 127.0.0.1:8080
# Username: admin
# Password: <from kubectl get secret response, without %>
argocd account update-password
```
